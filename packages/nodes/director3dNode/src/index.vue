<template>
  <nodeSkeleton v-bind="nodeProps" style="width: 320px">
    <button v-loading="modelLoading" type="button" class="directorContent nopan" :disabled="modelLoading" :title="modelError || undefined" aria-label="打开导演台" @dblclick.stop @click.stop="openEditor">
      <img v-if="preview" class="scenePreview" :src="preview" alt="最后镜头" draggable="false" />
      <div v-else class="emptyPreview">
        <icon-cube3d-sphere :size="38" stroke="1.2" />
      </div>
    </button>
  </nodeSkeleton>
  <sceneEditor v-if="editing" v-model:anchors="anchors" v-model:prompt="prompt" v-model:model="model" v-model:lighting="lighting" v-model:sceneSettings="sceneSettings"
    :scene="scene" :result="selectedPlan" :models="models" :modelsLoading="modelsLoading" :addingMannequin="addingMannequin"
    :plans="plans" :selectedPlanId="data.selectedPlanId ?? ''" :tasks="tasks"
    :exportingVideo="exportingVideo" :exportingImage="exportingImage" :exportProgress="exportProgress"
    @exportVideo="exportVideo" @exportImage="exportImage" @addMannequin="addMannequin"
    @selectPlan="data.selectedPlanId = $event" @loadModels="loadModels" @generate="generate()" @editInstruction="setPrompt" @close="editing = false">
    <template #input>
      <referenceItem v-if="refList.length" v-model="refList" @preview="setReferencePreview" @remove="removeReference" />
      <promptInput v-model="promptModel" v-model:text="prompt" :references="referenceMentions" />
    </template>
  </sceneEditor>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, ref, shallowRef, watch } from "vue";
import { useVueFlow } from "@vue-flow/core";
import { IconCube3dSphere } from "@tabler/icons-vue";
import { ElLoading, ElMessage } from "element-plus";
import { nodeSkeleton, nodeTools, useNode, useNodeFiles, z, type NodeHandle, type NodeData, type NodeAiModel } from "@toonflow/nodes-scaffold/runtime";
import { useNodeReferences } from "@toonflow/node-shared/useNodeReferences";
import promptInput from "@toonflow/node-shared/promptInput";
import referenceItem from "@toonflow/node-shared/referenceItem";
import sceneEditor from "./sceneEditor.vue";
import { capturePreview, sceneSchema, createEmptyScene, createMannequinObject, createStage, disposeStage, getSceneLighting, type LightingSettings, type SceneDocument, type SceneSettings } from "./scene";
import { createDirectorDraft } from "./agentTools";
import { directorPrompt } from "./agentPrompt";
import { anchorSchema, prepareMotion, sampleMotion, type CameraAnchor } from "./motion";
import { directorPlanSchema, prepareSceneAnimation, type DirectorPlan, type DirectorGeneration } from "./sceneAnimation";
import { renderImage, renderVideo } from "./renderMedia";

type PromptModel = NonNullable<InstanceType<typeof promptInput>["$props"]["modelValue"]>;
const modelDocumentSchema = z.strictObject({
  version: z.literal(1),
  scene: sceneSchema,
  plans: z.array(directorPlanSchema.safeExtend({ id: z.string().min(1), instruction: z.string().optional() })),
});
type ModelDocument = z.infer<typeof modelDocumentSchema>;
defineOptions({
  inheritAttrs: false,
  icon: IconCube3dSphere,
  handles: [{ id: "in", type: "target", dataType: ["STRING", "IMAGE", "VIDEO"], label: "文本、图片、视频输入" }] satisfies NodeHandle[],
});
const { node, nodeProps, previewReady, ai, files, nodeEvent } = useNode({ label: "3D导演台" });
const vLoading = ElLoading.directive;
const { addNodes, findNode, getNodes, nodeTypes, removeNodes } = useVueFlow();
const mediaFiles = useNodeFiles();
const exportingVideo = ref(false);
const exportingImage = ref("");
const exportProgress = ref(0);
const exportError = ref("");
const exportController = new AbortController();
nodeEvent.on("delete", () => { if (exportingVideo.value || exportingImage.value) throw new Error("正在导出，请完成后再删除导演节点"); });
const data = computed(() => node.data as typeof node.data & {
  modelPath?: string;
  modelSnapshot?: ModelDocument;
  lighting?: LightingSettings;
  sceneSettings?: SceneSettings;
  prompt?: string;
  promptModel?: PromptModel;
  model?: string;
  anchors?: CameraAnchor[];
  selectedPlanId?: string;
});
const { refList, referenceMentions, setReferencePreview, removeReference } = useNodeReferences();
const promptModel = computed({ get: () => data.value.promptModel ?? [], set: (value: PromptModel) => { data.value.promptModel = value; } });
const anchors = computed({ get: () => data.value.anchors ?? [], set: (value: CameraAnchor[]) => { data.value.anchors = value; } });
const prompt = computed({ get: () => data.value.prompt ?? "", set: (value: string) => { data.value.prompt = value; } });
const model = computed({ get: () => data.value.model ?? "", set: (value: string) => { data.value.model = value; } });
// ACT: 场景和动画只在节点内部持有，画布数据仅保存文件引用，避免拖动时遍历大量关键帧。
const modelDocument = shallowRef<ModelDocument>({ version: 1, scene: createEmptyScene(), plans: [] });
const plans = computed(() => modelDocument.value.plans);
const selectedPlan = computed(() => plans.value.find(plan => plan.id === data.value.selectedPlanId));
const scene = computed(() => modelDocument.value.scene);
const lighting = computed({ get: () => ({ ...getSceneLighting(scene.value), ...data.value.lighting }), set: (value: LightingSettings) => { data.value.lighting = value; } });
const sceneSettings = computed({ get: () => ({ gridVisible: true, skyVisible: true, ...data.value.sceneSettings }), set: (value: SceneSettings) => { data.value.sceneSettings = value; } });
const editing = ref(false);
const tasks = ref<DirectorGeneration[]>([]);
const modelsLoading = ref(false);
const models = ref<NodeAiModel[]>([]);
const selectedModel = computed(() => models.value.find(item => JSON.stringify([item.providerId, item.modelId]) === model.value));
const preview = ref("");
let previewVersion = 0;
const modelLoading = ref(true);
const modelError = ref("");
const addingMannequin = ref(false);
let modelSaving = Promise.resolve();
let disposed = false;
onBeforeUnmount(() => { disposed = true; exportController.abort(new Error("导演节点已关闭，导出已停止")); });

function getModelPath() {
  if (!node.id || /[\\/]/.test(node.id) || node.id === "." || node.id === "..") throw new Error("节点 ID 不能作为文件夹名称");
  return `assets/${node.id}/model.json`;
}

async function writeModel(workspaceFiles: ReturnType<typeof files.getWorkspaceFiles>, value: ModelDocument) {
  const path = getModelPath();
  for (const directory of ["assets", `assets/${node.id}`]) {
    await workspaceFiles.mkdir(directory).catch((error: { response?: { data?: { data?: { code?: string } } } }) => {
      if (error.response?.data?.data?.code !== "EEXIST") throw error;
    });
  }
  await workspaceFiles.writeJson(path, value);
  return path;
}

async function loadModel() {
  modelLoading.value = true;
  modelError.value = "";
  try {
    const { modelPath, modelSnapshot } = data.value;
    if (!modelPath && !modelSnapshot) return;
    const workspaceFiles = files.getWorkspaceFiles();
    if (modelPath && modelPath !== getModelPath()) throw new Error("导演台模型文件路径无效");
    const value = modelDocumentSchema.parse(modelSnapshot ?? await workspaceFiles.readJson(modelPath!));
    if (disposed) return;
    // 复制、跨画布粘贴时只传递一次快照，写入新节点目录成功后才移除快照。
    if (modelSnapshot) {
      const path = await writeModel(workspaceFiles, value);
      if (disposed) return;
      data.value.modelPath = path;
      delete data.value.modelSnapshot;
    }
    modelDocument.value = value;
    if (!selectedPlan.value) data.value.selectedPlanId = value.plans[0]?.id;
  } catch (error) {
    modelError.value = error instanceof Error ? error.message : "模型文件读取失败";
    if (!disposed) ElMessage.error(`导演台加载失败：${modelError.value}`);
  } finally {
    modelLoading.value = false;
  }
}

let modelReady = loadModel();
async function openEditor() {
  if (modelError.value) modelReady = loadModel();
  await modelReady;
  if (!disposed && !modelError.value) editing.value = true;
}

async function addMannequin() {
  if (modelLoading.value || modelError.value || addingMannequin.value || exportingVideo.value || exportingImage.value || tasks.value.some(task => !task.error)) return;
  addingMannequin.value = true;
  try {
    const workspaceFiles = files.getWorkspaceFiles();
    const saving = modelSaving.then(async () => {
      if (disposed) return;
      const current = scene.value;
      const object = createMannequinObject(crypto.randomUUID());
      const mannequins = current.objectList.filter(item => item.objType === "mannequin");
      object.position.x = mannequins.length ? Math.max(...mannequins.map(item => item.position.x)) + 1 : 0;
      const nextScene: SceneDocument = { ...current, objectList: [...current.objectList, object] };
      if (!current.objectList.length) {
        nextScene.sceneConfig = {
          ...current.sceneConfig,
          camera: { ...current.sceneConfig.camera, position: { x: 3, y: 2.2, z: 4 } },
          controls: { target: { x: 0, y: 0.9, z: 0 } },
        };
      }
      const value: ModelDocument = { ...modelDocument.value, scene: sceneSchema.parse(nextScene) };
      const path = await writeModel(workspaceFiles, value);
      if (disposed) return;
      modelDocument.value = value;
      data.value.modelPath = path;
    });
    modelSaving = saving.catch(() => {});
    await saving;
  } catch (error) {
    if (!disposed) ElMessage.error(error instanceof Error ? error.message : "人偶添加失败");
  } finally {
    addingMannequin.value = false;
  }
}

nodeEvent.on("copy", async () => {
  await nodeEvent.emit("save");
  if (modelError.value) throw new Error(`导演台模型未加载，无法复制：${modelError.value}`);
  return { modelPath: undefined, modelSnapshot: modelDocument.value };
});

nodeEvent.on("save", async (reason) => {
  await modelReady;
  let pending: Promise<void>;
  do {
    pending = modelSaving;
    await pending;
  } while (pending !== modelSaving);
  if (reason === "reload" && (exportingVideo.value || exportingImage.value || tasks.value.some(task => !task.error))) {
    throw new Error("导演台正在生成或导出，请完成后再刷新节点");
  }
});

async function exportToCanvas(kind: "image" | "video", key: string, aspect: number, render: (signal: AbortSignal) => Promise<File>, label = selectedPlan.value?.name ?? "视频") {
  if (kind === "video" ? exportingVideo.value : exportingImage.value) return;
  const type = `remote-${kind}Node`;
  if (!nodeTypes?.value?.[type]) return void ElMessage.error(`请先启用${kind === "image" ? "图片" : "视频"}节点插件`);
  if (kind === "video") { exportingVideo.value = true; exportProgress.value = 0; }
  else exportingImage.value = key;
  exportError.value = "";
  const id = crypto.randomUUID();
  let exportNode: ReturnType<typeof findNode<NodeData & { exportProgress?: number }>>;
  let discarded = false;
  let committed = false;
  let uploadStarted = false;
  let stopWatching = () => {};
  let workspaceFiles: ReturnType<typeof files.getWorkspaceFiles> | undefined;
  function addExportNode(label: string) {
    const width = 240 * aspect + 18;
    const x = node.computedPosition.x + (node.dimensions.width || 320) + 80;
    let y = node.computedPosition.y;
    for (const other of [...getNodes.value].sort((a, b) => a.computedPosition.y - b.computedPosition.y)) {
      const position = other.computedPosition;
      if (position.x < x + width && position.x + (other.dimensions.width || 320) > x && position.y < y + 320 && position.y + (other.dimensions.height || 300) > y) {
        y = position.y + (other.dimensions.height || 300) + 24;
      }
    }
    addNodes({ id, type, position: { x, y }, data: { label, outputs: {} } });
    return findNode(id)!;
  }
  try {
    workspaceFiles = files.getWorkspaceFiles();
    if (kind === "video") {
      exportNode = addExportNode(label);
      // ACT: 导出进度只在当前运行中使用，不能写入画布或复制到另一个节点。
      Object.defineProperty(exportNode.data, "exportProgress", { value: 0, writable: true, configurable: true });
      stopWatching = watch([() => findNode(id), exportProgress], ([current, progress]) => {
        if (current !== exportNode) discarded = true;
        if (!discarded) exportNode!.data.exportProgress = progress;
      }, { flush: "sync" });
    }
    const file = await render(exportController.signal);
    exportController.signal.throwIfAborted();
    if (discarded) return;
    uploadStarted = true;
    const path = await mediaFiles.uploadFile(id, file);
    exportController.signal.throwIfAborted();
    if (discarded) return;
    if (findNode(node.id) !== node || !nodeTypes?.value?.[type]) throw new Error("画布节点已变化，请重新导出");
    exportNode ??= addExportNode(file.name.replace(/\.[^.]+$/, ""));
    // 更新已有输出对象，让已挂载的视频节点及连接它的节点同步收到结果。
    (exportNode.data.outputs ??= {})[kind] = {
      dataType: kind === "image" ? "IMAGE" : "VIDEO", value: { url: path, mimeType: file.type },
    };
    committed = true;
    if (kind === "video") exportProgress.value = 100;
    ElMessage.success(`${kind === "image" ? "图片" : "视频"}已导出到画布`);
  } catch (error) {
    exportError.value = error instanceof Error ? error.message : "导出失败，请重试";
    if (!disposed && !discarded) ElMessage.error(exportError.value);
    if (exportNode && findNode(id) === exportNode) removeNodes(id);
  } finally {
    stopWatching();
    if (exportNode) delete exportNode.data.exportProgress;
    if (uploadStarted && !committed) {
      try { await workspaceFiles!.remove(`assets/${id}`, true); }
      catch (cleanupError) {
        const code = (cleanupError as { response?: { data?: { data?: { code?: string } } } })?.response?.data?.data?.code;
        if (!disposed && code !== "ENOENT") ElMessage.error("导出中断，临时素材清理失败");
      }
    }
    if (kind === "video") exportingVideo.value = false;
    else exportingImage.value = "";
  }
}

function exportVideo(aspect: number, plan = selectedPlan.value) {
  if (!plan) return;
  return exportToCanvas("video", "video", aspect, signal => renderVideo(scene.value, plan, aspect, signal, value => { exportProgress.value = value; }, lighting.value, sceneSettings.value), plan.name);
}

function exportImage(anchor: CameraAnchor | undefined, aspect: number, time: number, plan = selectedPlan.value) {
  const name = (anchor
    ? `关键帧${anchors.value.findIndex(item => item.id === anchor.id) + 1}`
    : `${plan?.name ?? "画面"}-${time.toFixed(1)}s`).replace(/[\\/:*?"<>|]/g, "-");
  return exportToCanvas("image", anchor?.id ?? `time-${time}`, aspect, async signal => {
    const file = await renderImage(scene.value, anchor, aspect, time, signal, plan, lighting.value, sceneSettings.value);
    return new File([file], `${name}.png`, { type: file.type });
  });
}

function waitModelsLoaded() {
  if (!modelsLoading.value) return Promise.resolve();
  return new Promise<void>(resolve => {
    const stop = watch(modelsLoading, value => { if (!value) { stop(); resolve(); } });
  });
}

async function loadModels() {
  if (modelsLoading.value) return;
  modelsLoading.value = true;
  try {
    const available = await ai.getModels();
    if (disposed) return;
    models.value = available;
    const first = models.value[0];
    // 供应商配置变化后旧选择可能不在当前列表，回退默认模型，避免生成与导出被卡住。
    if (model.value && !models.value.some(item => JSON.stringify([item.providerId, item.modelId]) === model.value)) {
      if (!disposed) ElMessage.warning("之前选择的生成模型已不可用，已切换为默认模型，请按需重选");
      model.value = first ? JSON.stringify([first.providerId, first.modelId]) : "";
    } else if (!model.value) {
      model.value = first ? JSON.stringify([first.providerId, first.modelId]) : "";
    }
  } catch (error) {
    if (!disposed) ElMessage.error(error instanceof Error ? error.message : "模型加载失败");
  } finally {
    modelsLoading.value = false;
  }
}

async function renderPreview(scene: SceneDocument, plan: DirectorPlan | undefined, lighting: LightingSettings, settings: SceneSettings) {
  const runtime = await createStage(document.createElement("canvas"), scene, undefined, undefined, lighting, settings);
  let player: ReturnType<typeof prepareSceneAnimation> | undefined;
  try {
    if (plan) {
      player = prepareSceneAnimation(runtime, plan);
      player.setTime(plan.duration);
      sampleMotion(runtime.camera, prepareMotion(plan.cameraFrames), plan.duration);
    }
    return capturePreview(runtime);
  } finally {
    player?.dispose();
    disposeStage(runtime);
  }
}

const planPreviews = new WeakMap<DirectorPlan, { scene: SceneDocument; lighting: LightingSettings; settings: SceneSettings; image: string }>();
watch([scene, selectedPlan, lighting, sceneSettings, modelLoading, previewReady], async ([value, plan, light, settings, loading, ready]) => {
  const version = ++previewVersion;
  preview.value = "";
  if (!ready || loading || modelError.value) return;
  try {
    const cached = plan && planPreviews.get(plan);
    const image = cached?.scene === value && JSON.stringify(cached.lighting) === JSON.stringify(light) && JSON.stringify(cached.settings) === JSON.stringify(settings) ? cached.image : await renderPreview(value, plan, light, settings);
    if (plan) planPreviews.set(plan, { scene: value, lighting: light, settings, image });
    if (!disposed && version === previewVersion) preview.value = image;
  } catch (error) {
    if (!disposed && version === previewVersion) ElMessage.error(error instanceof Error ? error.message : "预览生成失败");
  }
}, { immediate: true });

function setPrompt(value: string) {
  prompt.value = value;
  promptModel.value = value ? value.split("\n").map(text => [{ type: "Write", text }]) : [];
}

async function generate(requirement = prompt.value.trim()) {
  if (modelLoading.value || modelError.value || addingMannequin.value) return;
  const choice = selectedModel.value;
  if (!choice || !requirement) return;
  // 首次创建基础模型期间只接收一个请求，后续动画可并行生成。
  if (!scene.value.objectList.length && tasks.value.some(task => !task.error)) return;
  const baseScene = scene.value;
  const basePlanId = data.value.selectedPlanId ?? "";
  const task = { id: crypto.randomUUID(), instruction: requirement };
  tasks.value = [...tasks.value, task];
  setPrompt("");
  try {
    const workspaceFiles = files.getWorkspaceFiles();
    if (refList.value.some(item => item.value === undefined)) throw new Error("引用节点暂无内容，请先补充引用内容");
    const mediaReferences = refList.value
      .filter(item => item.value !== undefined && (item.dataType === "STRING" || item.dataType === "IMAGE" || item.dataType === "VIDEO"))
      .map(item => item.dataType === "STRING" ? { dataType: item.dataType, value: item.value } : { dataType: item.dataType, value: { ...item.value } });
    const references = anchors.value.map((value, index) => {
      const { camera, controls } = anchorSchema.parse(value);
      return { order: index + 1, camera, controls };
    });
    const basePlan = selectedPlan.value;
    const draft = createDirectorDraft(
      JSON.parse(JSON.stringify(baseScene)),
      basePlan ? directorPlanSchema.parse({ name: basePlan.name, duration: basePlan.duration, tracks: basePlan.tracks, cameraFrames: basePlan.cameraFrames }) : undefined,
      JSON.parse(JSON.stringify(plans.value)),
    );
    const directory = mediaReferences.some(item => item.dataType !== "STRING") ? (await workspaceFiles.list()).directory : undefined;
    if (disposed) return;
    await ai.generate({
      references: mediaReferences,
      providerId: choice.providerId,
      modelId: choice.modelId,
      systemPrompt: directorPrompt,
      prompt: JSON.stringify({ instruction: requirement, references, selectedPlanId: basePlanId, document: draft.summary() }),
      directory,
      tools: draft.tools,
    });
    if (disposed) return;
    if (!draft.edited) throw new Error("Agent 未修改方案，原方案未修改，请重试。");
    const result = draft.read();
    const nextScene = draft.sceneChanged ? result.scene : baseScene;
    const plan = { ...result.plan, id: task.id, instruction: requirement };
    // 工具仅编辑草稿；完成后验证实际渲染，成功落盘才替换当前方案。
    const previewLighting = { ...getSceneLighting(nextScene), ...data.value.lighting };
    const previewSettings = sceneSettings.value;
    const image = await renderPreview(nextScene, plan, previewLighting, previewSettings);
    if (disposed) return;
    // 动画请求可以并行生成，同一模型文件顺序保存，写入成功后才替换当前场景。
    const saving = modelSaving.then(async () => {
      if (disposed) return;
      if (scene.value !== baseScene) throw new Error("基础模型已被另一条指令更新，请基于当前模型重试。");
      const objects = new Map(nextScene.objectList.map(object => [object.threeJsonId, object]));
      const previousPlans = draft.sceneChanged
        ? plans.value.map(item => ({ ...item, tracks: item.tracks.filter(track => objects.has(track.objectId) && (!track.joint || objects.get(track.objectId)?.objType === "mannequin")) }))
        : plans.value;
      const value: ModelDocument = { version: 1, scene: nextScene, plans: [...previousPlans, plan] };
      const path = await writeModel(workspaceFiles, value);
      if (disposed) return;
      planPreviews.set(plan, { scene: nextScene, lighting: previewLighting, settings: previewSettings, image });
      modelDocument.value = value;
      data.value.modelPath = path;
      if ((data.value.selectedPlanId ?? "") === basePlanId) data.value.selectedPlanId = plan.id;
    });
    modelSaving = saving.catch(() => {});
    await saving;
    tasks.value = tasks.value.filter(item => item.id !== task.id);
  } catch (error) {
    if (!disposed) tasks.value = tasks.value.map(item => item.id === task.id ? { ...item, error: error instanceof Error ? error.message : "生成失败，请重试" } : item);
  }
}

nodeTools.register({
  name: "generate",
  description: "按当前或传入的指令生成导演台动画方案，等价于点击面板的“生成方案”按钮。instruction 缺省时使用节点输入框当前提示词，两者均为空、模型未配置或首次创建基础模型期间已有进行中任务时报错；生成会清空节点输入框。立即返回 generating，任务 id 留在 getGenerationStatus 的 generating 列表表示进行中，带 error 表示失败，从列表消失表示成功",
  parameters: z.strictObject({ instruction: z.string().max(4000).optional() }),
  async execute(args, { signal }) {
    signal?.throwIfAborted();
    if (modelLoading.value) throw new Error("模型文件加载中，请稍后再生成");
    if (modelError.value) throw new Error(`模型文件加载失败：${modelError.value}`);
    if (addingMannequin.value) throw new Error("正在添加人偶，请稍后再生成");
    if (modelsLoading.value) await waitModelsLoaded();
    if (!models.value.length) {
      void loadModels();
      await waitModelsLoaded();
    }
    if (!selectedModel.value) throw new Error(models.value.length ? "之前选择的生成模型不在当前可用列表，请重新选择" : "可用生成模型为空，请先在设置中配置模型供应商");
    const requirement = (args.instruction ?? prompt.value).trim();
    if (!requirement) throw new Error("instruction 与节点输入框提示词均为空");
    if (!scene.value.objectList.length && tasks.value.some(task => !task.error)) throw new Error("首次创建基础模型期间只接收一个生成任务，请等待完成");
    void generate(requirement);
    return { status: "generating" };
  },
});

nodeTools.register({
  name: "exportVideo",
  description: "把动画方案离线逐帧编码为 MP4 并在画布旁生成视频节点，等价于点击“导出视频节点”按钮。planId 缺省导出当前选中方案，可用 id 见 getGenerationStatus 返回的 plans，指定时不改变面板当前选中。本地渲染不调用媒体模型，页面保持打开即可、可在后台标签；总耗时约等于帧数乘单帧渲染耗时，与动画时长无关。立即返回 exporting，用 getGenerationStatus 查询 exportProgress 与完成状态",
  parameters: z.strictObject({
    planId: z.string().min(1).max(100).optional(),
    aspect: z.number().positive().max(4).optional(),
  }),
  execute(args, { signal }) {
    signal?.throwIfAborted();
    const plan = args.planId ? plans.value.find(item => item.id === args.planId) : selectedPlan.value;
    if (!plan) {
      if (!plans.value.length) throw new Error("没有任何动画方案，请先生成");
      if (args.planId) throw new Error(`方案不存在，可用方案：${plans.value.map(item => item.id).join("、")}`);
      throw new Error("没有选中的动画方案，可传 planId 指定或先生成、选中方案");
    }
    if (!nodeTypes?.value?.["remote-videoNode"]) throw new Error("请先启用视频节点插件");
    if (exportingVideo.value) throw new Error("视频正在导出，请等待完成");
    void exportVideo(args.aspect ?? 16 / 9, plan);
    return { status: "exporting" };
  },
});

nodeTools.register({
  name: "exportImage",
  description: "渲染指定画面为 PNG 并在画布旁生成图片节点，等价于点击关键帧上的导出按钮。anchorId 缺省导出第一个取景关键帧；没有关键帧时不报错，按 time（方案时间轴上的秒数，默认 0）用方案运镜画面导出。planId 缺省用当前选中方案，可用 id 见 getGenerationStatus 返回的 plans，指定时不改变面板当前选中。立即返回 exporting，用 getGenerationStatus 查询完成状态",
  parameters: z.strictObject({
    planId: z.string().min(1).max(100).optional(),
    anchorId: z.string().min(1).max(100).optional(),
    aspect: z.number().positive().max(4).optional(),
    time: z.number().min(0).max(300).optional(),
  }),
  execute(args, { signal }) {
    signal?.throwIfAborted();
    const plan = args.planId ? plans.value.find(item => item.id === args.planId) : selectedPlan.value;
    if (args.planId && !plan) {
      if (!plans.value.length) throw new Error("没有任何动画方案，请先生成");
      throw new Error(`方案不存在，可用方案：${plans.value.map(item => item.id).join("、")}`);
    }
    const anchor = args.anchorId ? anchors.value.find(item => item.id === args.anchorId) : anchors.value[0];
    if (args.anchorId && !anchor) {
      if (!anchors.value.length) throw new Error("没有取景关键帧，可省略 anchorId 按 time 导出方案画面");
      throw new Error(`关键帧不存在，可用关键帧：${anchors.value.map(item => item.id).join("、")}`);
    }
    if (!nodeTypes?.value?.["remote-imageNode"]) throw new Error("请先启用图片节点插件");
    if (exportingImage.value) throw new Error("图片正在导出，请等待完成");
    void exportImage(anchor, args.aspect ?? 16 / 9, args.time ?? anchor?.time ?? 0, plan);
    return { status: "exporting" };
  },
});

nodeTools.register({
  name: "getGenerationStatus",
  description: "查询导演台状态：generating 为生成任务列表（id 在列表=进行中，带 error=失败，从列表消失=成功）；exportingVideo/exportProgress 与 exportingImage 为导出进度，复位且无 exportError 即导出完成；anchors 列出可导出的关键帧，plans 列出全部方案及当前选中项",
  parameters: z.strictObject({}),
  execute(_args, { signal }) {
    signal?.throwIfAborted();
    return {
      generating: tasks.value,
      exportingVideo: exportingVideo.value,
      exportProgress: exportProgress.value,
      exportingImage: exportingImage.value,
      ...(exportError.value ? { exportError: exportError.value } : {}),
      ...(modelLoading.value ? { modelLoading: true } : {}),
      ...(modelError.value ? { modelError: modelError.value } : {}),
      selectedPlan: selectedPlan.value ? { id: selectedPlan.value.id, name: selectedPlan.value.name, duration: selectedPlan.value.duration } : null,
      plans: plans.value.map(({ id, name, duration }) => ({ id, name, duration })),
      anchors: anchors.value.map(({ id, time }) => ({ id, time })),
    };
  },
});

</script>

<style scoped lang="scss">
.directorContent {
  display: block;
  width: 100%;
  padding: 0;
  border: 0;
  cursor: pointer;
  aspect-ratio: 16 / 9;
  overflow: hidden;
  color: var(--el-text-color-regular);
  background: var(--el-fill-color-light);
  border-radius: var(--el-border-radius-base);
  &:focus-visible { outline: 2px solid var(--el-color-primary); outline-offset: 2px; }

  .scenePreview {
    display: block;
    width: 100%;
    height: 100%;
    object-fit: contain;
  }

  .emptyPreview {
    display: flex;
    height: 100%;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    color: var(--el-text-color-placeholder);
  }

}
</style>
