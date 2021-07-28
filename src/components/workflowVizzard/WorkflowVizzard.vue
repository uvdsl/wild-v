<template>
    <svg id="svgGraph" ref="svgRef" class="sizing" />
</template>

<script>
import * as d3 from "d3";
import { defineComponent } from "vue";
import { WILD } from "@/lib/namespaces";
export default defineComponent({
  name: "WorkflowVizzard",
  props: ["treeDataInput", "updateTreeData"],
  data() {
    return {
      id_node_counter: 0,
      duration: 500,
      offset: 100,
      svg: undefined,
      g_link_selection: undefined,
      g_node_selection: undefined,
      treemap: undefined,
      root: undefined,
      treeData: {
        label: "",
        children: [],
      },
    };
  },
  mounted() {
    this.treeData = this.treeDataInput;
    this.svg = d3.select("#svgGraph");

    this.svg.call(
      d3.zoom().on("zoom", () => {
        this.svg
          .selectAll(".nodes")
          .attr("transform", d3.event.transform.toString());
        this.svg
          .selectAll(".links")
          .attr("transform", d3.event.transform.toString());
      })
    );

    this.g_link_selection = this.svg
      .append("g")
      .attr("class", "links")
      .attr("stroke", "#ccc")
      .attr("stroke-width", 3)
      .attr("fill", "none")
      .selectAll("path");

    this.g_node_selection = this.svg
      .append("g")
      .attr("class", "nodes")
      .selectAll("g");

    const svg_pos = this.$refs.svgRef.getBoundingClientRect();
    // const svg_x_center = svg_pos.x + svg_pos.width / 2;
    // const svg_y_center = svg_pos.y + svg_pos.height / 2;

    // declares a tree layout and assigns the size
    this.treemap = d3.tree().size([svg_pos.height, svg_pos.width]);
    // Assigns parent, children, height, depth
    this.root = d3.hierarchy(this.treeData, (d) => d.children);
    this.root.x0 = svg_pos.width / 2;
    this.root.y0 = 0;

    this.update(this.root);
  },
  watch: {
    updateTreeData: function () {
      this.root = d3.hierarchy(this.root.data);
      this.update(this.updateTreeData);
    },
  },
  methods: {
    update(source) {
      // Assigns the x and y position for the nodes
      this.treeData = this.treemap(this.root);

      // Compute the new tree layout.
      const nodes = this.treeData.descendants();
      const links = this.treeData.descendants().slice(1);
      // Normalize for fixed-depth.
      nodes.forEach((d) => {
        d.y = d.depth * 200;
      });

      // ****************** Nodes section ***************************
      this.g_node_selection = this.g_node_selection
        .data(nodes, (d) => d.data.id || (d.data.id = this.id_node_counter++))
        .join(
          (enter) => {
            const selection = enter
              .append("g")
              .attr("cursor", "pointer")
              .on("click", this.select);
            selection
              .attr(
                "transform",
                () =>
                  "translate(" +
                  (source.y0 + this.offset) +
                  "," +
                  source.x0 +
                  ")"
              )
              .transition()
              .duration(this.duration)
              .attr(
                "transform",
                (d) => "translate(" + (d.y + this.offset) + "," + d.x + ")"
              );
            const width = 120;
            const height = 45;
            const border = 5;
            selection
              .append("rect")
              .attr("width", width)
              .attr("height", height)
              .attr("x", -(width + border) / 2)
              .attr("y", -(height + border) / 2)
              .attr("rx", "10px") // round corners
              .attr("fill", "#2E425E")
              .attr("stroke-width", border)
              .attr("stroke", this.color);

            selection
              .append("text")
              .text((d) => d.data.label)
              .attr("fill", "white")
              .style("text-anchor", "middle");

            return selection;
          },
          (update) => {
            update
              .transition()
              .duration(this.duration)
              .attr(
                "transform",
                (d) => "translate(" + (d.y + this.offset) + "," + d.x + ")"
              );
            update.select("rect").attr("stroke", this.color);
            update.select("text").text((d) => d.data.label);
            return update;
          },
          (exit) =>
            exit
              .lower()
              .transition()
              .duration(this.duration)
              .attr(
                "transform",
                () =>
                  "translate(" + (source.y + this.offset) + "," + source.x + ")"
              )
              .remove()
        );

      // ****************** links section ***************************

      this.g_link_selection = this.g_link_selection
        .data(links, (d) => d.parent.data.id + "->" + d.data.id)
        .join(
          (enter) => {
            const selection = enter.append("path");
            selection
              .attr("d", () => {
                const o = { x: source.x0, y: source.y0 };
                return this.diagonal(o, o);
              })
              .transition()
              .duration(this.duration)
              .attr("d", (d) => this.diagonal(d, d.parent));
            return selection;
          },
          (update) =>
            update
              .transition()
              .duration(this.duration)
              .attr("d", (d) => this.diagonal(d, d.parent)),
          (exit) =>
            exit
              .transition()
              .duration(this.duration)
              .attr("d", () => {
                const o = { x: source.x, y: source.y };
                return this.diagonal(o, o);
              })
              .remove()
        );

      // Store the old positions for transition.
      nodes.forEach((d) => {
        d.x0 = d.x;
        d.y0 = d.y;
      });
    },
    // Creates a curved (diagonal) path from parent to the child nodes
    diagonal(s, d) {
      const path = `M ${s.y + this.offset} ${s.x} 
            C ${(s.y + d.y) / 2 + this.offset} ${s.x},
              ${(s.y + d.y) / 2 + this.offset} ${d.x},
              ${d.y + this.offset} ${d.x}`;
      return path;
    },
    color(d) {
      // if collapsed, then lightsteelblue, else white
      switch (d.data.type) {
        case WILD("SequentialActivity"):
          return "var(--teal-300)";
        case WILD("ParallelActivity"):
          return "var(--cyan-300)";
        case WILD("ConditionalActivity"):
          return "var(--indigo-300)";
        case WILD("InterfaceActivity"):
          return "var(--pink-300)";
        case WILD("AtomicActivity"):
          return "var(--primary-color)";
        default:
          return "white";
      }
    },
    // add children on click.
    select(d) {
      this.$emit("select", d);
    },
  },
});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>

</style>
