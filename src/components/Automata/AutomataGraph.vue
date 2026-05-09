<template>
  <div class="graph-wrap">
    <div ref="graphEl" class="graph"></div>

    <div class="graph__legend" aria-label="Legend">
      <span class="graph__legend-item">
        <span class="graph__legend-arrow">→</span>
        <span>start</span>
      </span>
      <span class="graph__legend-item">
        <span class="graph__legend-swatch graph__legend-swatch--state"></span>
        <span>state</span>
      </span>
      <span class="graph__legend-item">
        <span class="graph__legend-swatch graph__legend-swatch--final"></span>
        <span>final</span>
      </span>
      <span class="graph__legend-item">
        <span class="graph__legend-swatch graph__legend-swatch--active"></span>
        <span>active</span>
      </span>
      <span class="graph__legend-item">
        <span class="graph__legend-edge"></span>
        <span>active path</span>
      </span>
    </div>

    <button
      class="graph__reset"
      type="button"
      @click="recenter"
      title="Recenter graph"
    >
      <svg viewBox="0 0 16 16" width="14" height="14" aria-hidden="true">
        <circle cx="8" cy="8" r="6" stroke="currentColor" stroke-width="1.4" fill="none" />
        <path
          d="M8 4 V8 L11 10"
          stroke="currentColor"
          stroke-width="1.6"
          fill="none"
          stroke-linecap="round"
        />
      </svg>
      <span>Recenter</span>
    </button>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch } from "vue";
import { Network, DataSet } from "vis-network/standalone";

const props = defineProps({
  automata: { type: Object, required: true },
  activeStates: { type: Array, default: () => [] },
  activeEdges: { type: Array, default: () => [] },
});

const graphEl = ref(null);
let network = null;
let nodesDataSet = null;
let edgesDataSet = null;
let lastStructureKey = "";

const palette = {
  stateBg: "rgba(255, 255, 255, 0.04)",
  stateBorder: "#5b6781",
  finalBorder: "#06b6d4",
  finalGlow: "rgba(6, 182, 212, 0.45)",
  activeBg: "#f59e0b",
  activeBorder: "#fbbf24",
  activeText: "#0a0e1a",
  text: "#e5edf7",
  edge: "#7c8ba8",
  edgeText: "#e5edf7",
  edgeBg: "rgba(10, 14, 26, 0.9)",
  edgeActive: "#fbbf24",
  edgeActiveLabelBg: "rgba(245, 158, 11, 0.18)",
  edgeActiveText: "#fde68a",
  startEdge: "#06b6d4",
};

const START_ID = "__start__";

const buildStructureKey = (automata) => {
  const parts = automata.states.map((tList, i) => {
    const ts = tList.map((t) => `${t.value}>${t.to}`).join("|");
    return `${i}:${ts}`;
  });
  return (
    parts.join(";") +
    "@" +
    [...automata.finalStates].sort((a, b) => a - b).join(",") +
    "#" +
    [...automata.alphabets].sort().join(",")
  );
};

const nodeStyle = (isFinal, isActive) => {
  if (isActive) {
    return {
      background: palette.activeBg,
      border: palette.activeBorder,
      fontColor: palette.activeText,
      borderWidth: isFinal ? 4 : 2,
      shadow: {
        enabled: true,
        color: "rgba(245, 158, 11, 0.6)",
        size: 22,
        x: 0,
        y: 0,
      },
    };
  }
  if (isFinal) {
    return {
      background: palette.stateBg,
      border: palette.finalBorder,
      fontColor: palette.text,
      borderWidth: 4,
      shadow: { enabled: true, color: palette.finalGlow, size: 14, x: 0, y: 0 },
    };
  }
  return {
    background: palette.stateBg,
    border: palette.stateBorder,
    fontColor: palette.text,
    borderWidth: 2,
    shadow: { enabled: false },
  };
};

const buildNodes = () => {
  const automata = props.automata;
  const activeSet = new Set(props.activeStates);

  const nodes = automata.states.map((_, index) => {
    const isFinal = automata.finalStates.includes(index);
    const isActive = activeSet.has(index);
    const style = nodeStyle(isFinal, isActive);
    return {
      id: index,
      label: "S" + index,
      shape: "circle",
      borderWidth: style.borderWidth,
      borderWidthSelected: style.borderWidth,
      color: {
        background: style.background,
        border: style.border,
        hover: { background: style.background, border: style.border },
        highlight: { background: style.background, border: style.border },
      },
      font: {
        face: "JetBrains Mono",
        color: style.fontColor,
        size: 16,
        bold: {
          color: style.fontColor,
          size: 16,
          mod: "bold",
          face: "JetBrains Mono",
        },
      },
      margin: 14,
      shadow: style.shadow,
    };
  });

  // Phantom "start" node so users see an arrow into S0.
  // It has no physics so it stays parked next to S0 after stabilization.
  nodes.unshift({
    id: START_ID,
    label: "",
    shape: "dot",
    size: 1,
    color: {
      background: "rgba(0,0,0,0)",
      border: "rgba(0,0,0,0)",
      hover: { background: "rgba(0,0,0,0)", border: "rgba(0,0,0,0)" },
      highlight: { background: "rgba(0,0,0,0)", border: "rgba(0,0,0,0)" },
    },
    physics: false,
    fixed: false,
    chosen: false,
  });

  return nodes;
};

const edgeKey = (from, to, value) => `${from}->${to}#${value}`;

const buildEdges = () => {
  const automata = props.automata;
  const activeKeys = new Set(
    (props.activeEdges || []).map((e) => edgeKey(e.from, e.to, e.value))
  );

  const edges = [
    {
      id: `${START_ID}->0`,
      from: START_ID,
      to: 0,
      arrows: { to: { enabled: true, scaleFactor: 0.7 } },
      color: { color: palette.startEdge, inherit: false },
      width: 2,
      smooth: { enabled: true, type: "continuous", roundness: 0 },
      dashes: false,
      label: "",
      physics: false,
      length: 90,
    },
  ];

  for (let from = 0; from < automata.states.length; from++) {
    const grouped = new Map();
    for (const t of automata.states[from]) {
      const k = `${from}->${t.to}`;
      if (!grouped.has(k)) grouped.set(k, { from, to: t.to, values: [] });
      grouped.get(k).values.push(t.value);
    }

    let i = 0;
    for (const { to, values } of grouped.values()) {
      const isLoop = from === to;
      const isActive = values.some((v) =>
        activeKeys.has(edgeKey(from, to, v))
      );

      edges.push({
        id: `${from}-${to}-${i++}`,
        from,
        to,
        label: values.join(", "),
        arrows: {
          to: {
            enabled: true,
            scaleFactor: isActive ? 1 : 0.7,
            type: "arrow",
          },
        },
        color: {
          color: isActive ? palette.edgeActive : palette.edge,
          hover: palette.edgeActive,
          highlight: palette.edgeActive,
          inherit: false,
        },
        width: isActive ? 4 : 1.6,
        selectionWidth: 0,
        smooth: isLoop
          ? { enabled: true, type: "curvedCW", roundness: 0.7 }
          : { enabled: true, type: "dynamic", roundness: 0.4 },
        dashes: false,
        font: {
          face: "JetBrains Mono",
          color: isActive ? palette.edgeActiveText : palette.edgeText,
          background: isActive ? palette.edgeActiveLabelBg : palette.edgeBg,
          strokeWidth: 0,
          size: isActive ? 16 : 13,
          bold: isActive,
        },
        shadow: isActive
          ? {
              enabled: true,
              color: "rgba(245, 158, 11, 0.7)",
              size: 14,
              x: 0,
              y: 0,
            }
          : { enabled: false },
      });
    }
  }
  return edges;
};

const baseOptions = {
  layout: { improvedLayout: true },
  physics: {
    enabled: true,
    barnesHut: {
      gravitationalConstant: -3500,
      springLength: 150,
      springConstant: 0.05,
      damping: 0.6,
      avoidOverlap: 0.5,
    },
    stabilization: {
      enabled: true,
      iterations: 250,
      updateInterval: 25,
      fit: true,
    },
  },
  interaction: {
    dragNodes: false,
    dragView: true,
    zoomView: true,
    hover: true,
    multiselect: false,
    selectable: false,
    keyboard: false,
    tooltipDelay: 200,
  },
  nodes: { shape: "circle", size: 24 },
  edges: { arrows: { to: { scaleFactor: 0.7 } } },
};

const computeBounds = () => {
  if (!network) return null;
  const positions = network.getPositions();
  const ids = Object.keys(positions).filter((id) => id !== START_ID);
  if (!ids.length) return null;
  let minX = Infinity,
    maxX = -Infinity,
    minY = Infinity,
    maxY = -Infinity;
  for (const id of ids) {
    const p = positions[id];
    if (p.x < minX) minX = p.x;
    if (p.x > maxX) maxX = p.x;
    if (p.y < minY) minY = p.y;
    if (p.y > maxY) maxY = p.y;
  }
  return { minX, maxX, minY, maxY };
};

const clampView = (animate = true) => {
  if (!network) return;
  const bounds = computeBounds();
  if (!bounds) return;

  const scale = network.getScale();
  const view = network.getViewPosition();
  const canvas = network.canvas.frame.canvas;
  const halfW = canvas.clientWidth / scale / 2;
  const halfH = canvas.clientHeight / scale / 2;

  const margin = 60 / scale;
  let nx = view.x;
  let ny = view.y;

  if (view.x - halfW > bounds.maxX - margin) nx = bounds.maxX - margin + halfW;
  if (view.x + halfW < bounds.minX + margin) nx = bounds.minX + margin - halfW;
  if (view.y - halfH > bounds.maxY - margin) ny = bounds.maxY - margin + halfH;
  if (view.y + halfH < bounds.minY + margin) ny = bounds.minY + margin - halfH;

  if (nx !== view.x || ny !== view.y) {
    network.moveTo({
      position: { x: nx, y: ny },
      scale,
      animation: animate ? { duration: 220, easingFunction: "easeOutQuad" } : false,
    });
  }
};

const recenter = () => {
  if (!network) return;
  network.fit({ animation: { duration: 350, easingFunction: "easeInOutQuad" } });
};

// Park the phantom start node next to S0 once positions exist.
const placeStartNode = () => {
  if (!network) return;
  const positions = network.getPositions([0]);
  const s0 = positions[0];
  if (s0) {
    network.moveNode(START_ID, s0.x - 95, s0.y);
  }
};

const updateHighlightOnly = () => {
  if (!nodesDataSet || !edgesDataSet) return;
  const automata = props.automata;
  const activeSet = new Set(props.activeStates);

  const nodeUpdates = automata.states.map((_, index) => {
    const isFinal = automata.finalStates.includes(index);
    const isActive = activeSet.has(index);
    const style = nodeStyle(isFinal, isActive);
    return {
      id: index,
      borderWidth: style.borderWidth,
      borderWidthSelected: style.borderWidth,
      color: {
        background: style.background,
        border: style.border,
        hover: { background: style.background, border: style.border },
        highlight: { background: style.background, border: style.border },
      },
      font: {
        face: "JetBrains Mono",
        color: style.fontColor,
        size: 16,
        bold: {
          color: style.fontColor,
          size: 16,
          mod: "bold",
          face: "JetBrains Mono",
        },
      },
      shadow: style.shadow,
    };
  });
  nodesDataSet.update(nodeUpdates);

  // Wholesale replace edges so the active styling on labels/colors/shadows
  // can't get partially-merged with stale values.
  const newEdges = buildEdges();
  edgesDataSet.clear();
  edgesDataSet.add(newEdges);
};

const renderNetwork = () => {
  const nodes = buildNodes();
  const edges = buildEdges();

  if (network) {
    network.destroy();
    network = null;
  }

  nodesDataSet = new DataSet(nodes);
  edgesDataSet = new DataSet(edges);

  network = new Network(
    graphEl.value,
    { nodes: nodesDataSet, edges: edgesDataSet },
    baseOptions
  );

  network.on("dragEnd", () => clampView(true));
  network.on("zoom", () => clampView(false));

  network.once("stabilizationIterationsDone", () => {
    network.setOptions({ physics: false });
    placeStartNode();
    network.fit({ animation: false });
  });
};

onMounted(() => {
  lastStructureKey = buildStructureKey(props.automata);
  renderNetwork();
});

onBeforeUnmount(() => {
  if (network) {
    network.destroy();
    network = null;
  }
});

watch(
  () => props.automata,
  (newVal) => {
    if (!newVal) return;
    const newKey = buildStructureKey(newVal);
    if (newKey !== lastStructureKey) {
      lastStructureKey = newKey;
      renderNetwork();
    } else {
      updateHighlightOnly();
    }
  },
  { deep: true }
);

watch(
  () => [props.activeStates, props.activeEdges],
  () => {
    updateHighlightOnly();
  },
  { deep: true }
);
</script>

<style scoped lang="scss">
.graph-wrap {
  position: relative;
  width: 100%;
  height: 100%;
  border-radius: 1.2rem;
  overflow: hidden;
}

.graph {
  width: 100%;
  height: 100%;
  background: var(--bg-primary);
  border: none;
}

.graph__legend {
  position: absolute;
  top: 1rem;
  left: 1rem;
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.6rem 0.9rem;
  background: rgba(20, 25, 38, 0.7);
  backdrop-filter: blur(6px);
  border: 0.1rem solid var(--border);
  border-radius: 0.6rem;
  font-size: 1.1rem;
  font-family: var(--font-ui);
  font-weight: 500;
  color: var(--text-secondary);
  z-index: 5;
  pointer-events: none;
  flex-wrap: wrap;
  max-width: calc(100% - 11rem);
}

.graph__legend-item {
  display: inline-flex;
  align-items: center;
  gap: 0.45rem;
  white-space: nowrap;
}

.graph__legend-swatch {
  width: 1.2rem;
  height: 1.2rem;
  border-radius: 50%;
  display: inline-block;

  &--state {
    background: rgba(255, 255, 255, 0.04);
    border: 0.15rem solid #5b6781;
  }
  &--final {
    background: rgba(255, 255, 255, 0.04);
    border: 0.3rem solid var(--accent);
    box-shadow: 0 0 0.4rem rgba(6, 182, 212, 0.5);
  }
  &--active {
    background: var(--warning);
    border: 0.15rem solid #fbbf24;
    box-shadow: 0 0 0.5rem rgba(245, 158, 11, 0.6);
  }
}

.graph__legend-arrow {
  color: var(--accent);
  font-weight: 700;
  font-size: 1.4rem;
  line-height: 1;
}

.graph__legend-edge {
  width: 1.6rem;
  height: 0.3rem;
  background: var(--warning);
  border-radius: 0.2rem;
  display: inline-block;
  box-shadow: 0 0 0.5rem rgba(245, 158, 11, 0.7);
}

.graph__reset {
  position: absolute;
  top: 1rem;
  right: 1rem;
  display: inline-flex;
  align-items: center;
  gap: 0.6rem;
  padding: 0.6rem 1rem;
  background: rgba(20, 25, 38, 0.7);
  backdrop-filter: blur(6px);
  border: 0.1rem solid var(--border);
  border-radius: 0.6rem;
  color: var(--text-secondary);
  font-size: 1.2rem;
  font-weight: 500;
  font-family: var(--font-ui);
  transition: all 0.15s;
  z-index: 5;

  &:hover {
    color: var(--accent);
    border-color: var(--accent);
    background: var(--bg-elevated);
  }
}
</style>
