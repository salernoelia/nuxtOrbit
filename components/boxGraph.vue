<template>
  <div class="main">
    <div class="filterBox">
      
    <button class="filterButton"
        v-for="ecosystem in ecosystems"
        :key="ecosystem"
        @click="toggleEcosystem(ecosystem)"
        :class="{ active: filter.includes(ecosystem) }"
      >
        {{ ecosystem }}
      </button>

    </div>

    <div class="canvas" ref="canvas"></div>
  </div>
</template>

<style>
body {
  top: 0;
  overflow: hidden;
  overflow-y: hidden;
  overflow-x: hidden;
}

.filterBox {
  display: flex;
  flex-direction: row;
  position: fixed;
  min-width: 300px;
  top: 0;
  left: 0;
  z-index: 2;
  pointer-events: none;
}

.filterButton {
  z-index: 3;
  margin: 10px;
  pointer-events: auto;
 
}

.canvas {
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  position: absolute;
}

.main {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  width: 100%;
  position: relative;
}
</style>

<script>
import * as d3 from "d3";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import data from "~/static/data/data.json";


export default {
  data() {
    return {
      data,
      ecosystems: [], // Array to store unique ecosystems
      filter: [] // Array to store active filters
    };
  },
  mounted() {
    
    this.ecosystems = Array.from(new Set(this.data.map((habitat) => habitat.Habitat_name)));
    
    // Select the canvas element
    const canvas = this.$refs.canvas;
    
    console.log(this.data[0].Area_Habitat);
    const data = {
  name: "Root",
  children: this.data.map((habitat) => ({
    name: habitat.Habitat_name,
    value: parseFloat(habitat.Area_Habitat),
  })),
};

    // Set up dimensions
    const width = 400;
    const height = 400;

    // Create SVG container
    const svg = d3
      .select("body")
      .append("svg")
      .attr("width", width)
      .attr("height", height);

    // Create a hierarchical layout
    const root = d3.hierarchy(data).sum((d) => d.value);

    // Create color scale
    const color = d3.scaleOrdinal(d3.schemeCategory10);

    const partition = d3
      .partition()
      .size([width, height])
      .padding(1)
      .round(true);

    // Apply the layout to our data
    const nodes = partition(root).descendants();

    // Draw rectangles with varying sizes and colors
    svg
      .selectAll("rect")
      .data(nodes.filter((d) => !this.filter.length || this.filter.includes(d.data.name)))
      .enter()
      .append("rect")
      .attr("x", (d) => d.x0)
      .attr("y", (d) => d.y0)
      .attr("fill", (d) => color(d.depth))
      .attr("width", (d) => d.x1 - d.x0)
      .attr("height", (d) => d.y1 - d.y0)
      .attr("stroke", "white")
      .attr("stroke-width", 2);

    const colors = [0x2194ce, 0xff6347, 0x8a2be2, 0x00fa9a, 0xffd700];

    // Read out d3.js generated rectangles and import them to Three.js
    const rectangles = svg.selectAll("rect").nodes();

    // Create a Three.js scene
    const scene = new THREE.Scene();

    // Add axes helpers
    const axesHelper = new THREE.AxesHelper(100);

    
    scene.add(axesHelper);

    rectangles.forEach((rectangle, index) => {
      let rectWidth = parseFloat(rectangle.getAttribute("width")) / 6;
      let rectHeight = parseFloat(rectangle.getAttribute("height")) / 6;

      // Generate a random extrusion depth between 10 and 30
      const rectDepth = Math.random() * (30 - 10) + 10;

      // Extrude in the positive direction
      const geometryPositive = new THREE.ExtrudeGeometry(
        new THREE.Shape([
          new THREE.Vector2(0, 0),
          new THREE.Vector2(rectWidth, 0),
          new THREE.Vector2(rectWidth, rectHeight),
          new THREE.Vector2(0, rectHeight),
          new THREE.Vector2(0, 0),
        ]),
        { depth: rectDepth, bevelEnabled: false }
      );

      // Extrude in the negative direction
      const geometryNegative = new THREE.ExtrudeGeometry(
        new THREE.Shape([
          new THREE.Vector2(0, 0),
          new THREE.Vector2(rectWidth, 0),
          new THREE.Vector2(rectWidth, rectHeight),
          new THREE.Vector2(0, rectHeight),
          new THREE.Vector2(0, 0),
        ]),
        { depth: Math.abs(rectDepth), bevelEnabled: false }
      );

      // Create meshes for positive and negative extrusions
      const material = new THREE.MeshPhongMaterial({
        color: colors[index % colors.length],
      });
      const meshPositive = new THREE.Mesh(geometryPositive, material);
      const meshNegative = new THREE.Mesh(geometryNegative, material);

      // Position the meshes to "kiss" each other
      meshPositive.position.set(
        (parseFloat(rectangle.getAttribute("x")) + rectHeight) / 6 - 40,
        -(parseFloat(rectangle.getAttribute("y")) + rectHeight / 2) / 6,
        0
      );

      meshNegative.position.set(
        (parseFloat(rectangle.getAttribute("x")) + rectHeight) / 6 - 40,
        -(parseFloat(rectangle.getAttribute("y")) + rectHeight / 2) / 6,
        -rectDepth
      );

      // Add meshes to the scene
      scene.add(meshPositive);
      scene.add(meshNegative);
    });
    scene.background = new THREE.Color(0xffffff); // Set background color to white

    // Add ambient light with increased intensity and white color
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.8); // Increase intensity and set color to white
    scene.add(ambientLight);

    // Add directional light with increased intensity and white color
    const directionalLight = new THREE.DirectionalLight(0xffffff, 1.2); // Increase intensity and set color to white
    directionalLight.position.set(1, 1, 1).normalize(); // Adjust position
    scene.add(directionalLight);

    // Create a camera
    const camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
      0.1,
      1000
    );
    camera.position.set(0, 0, 200); // Adjust camera position

    // Rotate the scene so it is flat on X and Y axis
    scene.rotation.x = -Math.PI / 3;

    // Create three.js renderer
    const renderer = new THREE.WebGLRenderer({ antialias: true });

    // Set the pixel ratio for better quality on high-density displays
    renderer.setPixelRatio(window.devicePixelRatio);

    renderer.setSize(window.innerWidth, window.innerHeight);
    canvas.appendChild(renderer.domElement);

    // Create OrbitControls
    // Set up OrbitControls with additional features
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true; // an animation loop is required when either damping or auto-rotation are enabled
    controls.dampingFactor = 0.25; // setting damping factor for smoother motion
    controls.screenSpacePanning = false; // enable/disable screen-space panning
    controls.maxPolarAngle = Math.PI / 2; // limit the polar angle to avoid going upside down

    controls.minPolarAngle = 0; // Adjust as needed
    controls.maxPolarAngle = Math.PI; // Adjust as needed

    // Animation loop
    const animate = function () {
      requestAnimationFrame(animate);
      renderer.render(scene, camera);
      // Update the OrbitControls
      controls.update();
    };

    animate();

    camera.lookAt(new THREE.Vector3(0, 0, 0)); // Look at the center of the scene
  },
  methods: {
  toggleEcosystem(ecosystem) {
    const index = this.filter.indexOf(ecosystem);
    if (index === -1) {
      this.filter.push(ecosystem);
    } else {
      this.filter.splice(index, 1);
    }
    this.updateGraph();
  },
  updateGraph() {
    d3.select("svg").selectAll("rect").remove();
  },
}
};
</script>

<style lang="scss" scoped></style>
