<template>
  <div>
    <div class="graph-container-big" id="graph"></div>
  </div>
</template>

<script>
import * as d3 from "d3-selection";
require("d3-graphviz");
import link from "@/assets/config/link.json";
import tree from "@/assets/config/tree.json";
const colors = [
  "#bbdefb",
  "#90caf9",
  "#64b5f6",
  "#42a5f5",
  "#2196f3",
  "#1e88e5",
  "#1976d2"
];

export default {
  data() {
    return {
      allIdcs: [],
      selectedIdc: "0022_0000000001",
      tabList: [],
      payload: {
        filters: [],
        pageable: {
          pageSize: 10,
          startIndex: 0
        },
        paging: true
      },
      graph: new Map(),
      graphBig: "",
      layerId: 5,
      idcDesignData: null,
      zoneLinkDesignData: new Map(),
      currentTab: "resource-design",
      currentGraph: "",
      spinShow: false,
      isDataChanged: false
    };
  },
  mounted() {
    this.idcDesignData = tree.data[0];
    this.getZoneLink();
  },
  methods: {
    getZoneLink() {
      this.zoneLinkDesignData = new Map();
      const { data } = link;
      const idcLink = data.find(_ => _.idcGuid === this.selectedIdc);
      if (idcLink && idcLink.linkList) {
        idcLink.linkList.forEach(_ => {
          let zoneLink = {};
          if (
            _.data.zone_design1.idc_design === this.selectedIdc &&
            _.data.zone_design2.idc_design === this.selectedIdc
          ) {
            zoneLink.azone = `g_${_.data.zone_design1.guid}`;
            zoneLink.bzone = `g_${_.data.zone_design2.guid}`;
            const guid = this.idcDesignData.data.guid;
            if (this.zoneLinkDesignData.has(guid)) {
              this.zoneLinkDesignData.get(guid).push(zoneLink);
            } else {
              this.zoneLinkDesignData.set(guid, [zoneLink]);
            }
          }
        });
      }
      this.initGraph();
    },
    initGraph() {
      let graph;
      graph = d3.select("#graph");
      graph
        .on("dblclick.zoom", null)
        .on("wheel.zoom", null)
        .on("mousewheel.zoom", null);
      let graphZoom = graph
        .graphviz()
        .width(window.innerWidth * 0.96)
        .height(window.innerHeight * 1.2)
        .zoom(true);
      if (this.graph.has(this.idcDesignData.guid)) {
        this.graph[this.idcDesignData.guid] = graphZoom;
      } else {
        this.graph.set(this.idcDesignData.guid, graphZoom);
      }
      this.renderGraph(this.idcDesignData);
      this.spinShow = false;
    },
    renderGraph(idcData) {
      let nodesString = this.genDOT(
        idcData,
        this.zoneLinkDesignData.get(idcData.guid)
      );
      this.graph.get(idcData.guid).renderDot(nodesString);
      let fsize = 16;
      let divWidth = window.innerWidth,
        divHeight = window.innerHeight;
      let children = idcData.children || [];
      let svg = d3.select("#graph").select("svg");
      svg.attr("width", divWidth).attr("height", divHeight);
      svg.attr("viewBox", "0 0 " + divWidth + " " + divHeight);

      children.forEach(zone => {
        d3.select(`#g_${zone.guid}`)
          .select("polygon")
          .attr("fill", colors[0]);
        if (Array.isArray(zone.children)) {
          let points = d3
            .select("#g_" + zone.guid)
            .select("polygon")
            .attr("points")
            .split(" ");
          let p = {
            x: parseInt(points[1].split(",")[0]),
            y: parseInt(points[1].split(",")[1])
          };
          let pw = parseInt(points[0].split(",")[0] - points[1].split(",")[0]);
          let ph = parseInt(points[2].split(",")[1] - points[1].split(",")[1]);
          this.setChildren(zone, p, pw, ph, fsize, 1, idcData.guid);
        }
      });
    },
    setChildren(node, p1, pw, ph, tfsize, deep, idcName) {
      let graph;
      if (idcName === "graphBig") {
        graph = d3.select("#graphBig").select("#g_" + node.guid);
      } else {
        graph = d3.select("#graph").select("#g_" + node.guid);
      }
      let n = node.children.length;
      let w, h, mgap, fontsize, strokewidth;
      let rx, ry, tx, ty, g;
      let color = colors[deep];
      if (pw > ph * 1.2) {
        if (pw / n > ph - tfsize) {
          mgap = (ph - tfsize) * 0.04;
          fontsize =
            tfsize * 0.8 > (ph - tfsize) * 0.1
              ? (ph - tfsize) * 0.1
              : tfsize * 0.8;
          fontsize =  fontsize < 8 ? 8: fontsize
          strokewidth = (ph - tfsize) * 0.005;
        } else {
          mgap = (pw / n) * 0.04;
          fontsize =
            tfsize * 0.8 > (pw / n) * 0.1 ? (pw / n) * 0.1 : tfsize * 0.8;
          fontsize =  fontsize < 8 ? 8: fontsize
          strokewidth = (pw / n) * 0.005;
        }
        w = (pw - mgap) / n - mgap;
        h = ph - tfsize - 2 * mgap;
        for (var i = 0; i < n; i++) {
          rx = p1.x + (w + mgap) * i + mgap;
          ry = p1.y + tfsize + mgap;
          tx = p1.x + (w + mgap) * i + w * 0.5 + mgap;
          if (Array.isArray(node.children[i].children)) {
            ty = p1.y + tfsize + mgap + fontsize;
          } else {
            ty = p1.y + tfsize + mgap + h * 0.5;
          }

          g = graph
            .append("g")
            .attr("class", "node")
            .attr("id", "g_" + node.children[i].guid);
          g.append("rect")
            .attr("x", rx)
            .attr("y", ry)
            .attr("ry", 20-deep*4)
            .attr("ry", 20-deep*4)
            .attr("width", w)
            .attr("height", h)
            .attr("stroke", "black")
            .attr("fill", color)
            .attr("stroke-width", strokewidth);

          /***********customize-start***************/
          if (node.isLast) {
            g.append("circle")
            .attr("cx", tx)
            .attr("cy", ty+12)
            .attr("r", 10)
            .attr("stroke", "blue")
            .attr("fill", color)
            .on("dbclick", this.dbClick(11))
            .attr("stroke-width", strokewidth);
            g.append("text")
            .attr("x", tx)
            .attr("y", ty+16)
            .text('12%')
            .attr("style", "text-anchor:middle")
            .attr("font-size", fontsize);
          }
          /***********customize-end***************/
          g.append("text")
            .attr("x", tx)
            .attr("y", ty)
            .text(
              this.test(node,node.children[i])
              // node.children[i].data.code
              //   ? node.children[i].data.code
              //   : node.children[i].data.key_name
            )
            .attr("style", "text-anchor:middle")
            .attr("font-size", fontsize);
          if (Array.isArray(node.children[i].children)) {
            this.setChildren(
              node.children[i],
              { x: rx, y: ry },
              w,
              h,
              fontsize,
              deep + 1,
              idcName
            );
          }
        }
      } else {
        if ((ph - tfsize) / n > pw) {
          mgap = pw * 0.04;
          fontsize = tfsize * 0.8 > pw * 0.1 ? pw * 0.1 : tfsize * 0.8;
          strokewidth = pw * 0.005;
        } else {
          mgap = ((ph - tfsize) / n) * 0.04;
          fontsize =
            tfsize * 0.8 > ((ph - tfsize) / n) * 0.1
              ? ((ph - tfsize) / n) * 0.1
              : tfsize * 0.8;
          strokewidth = ((ph - tfsize) / n) * 0.005;
        }

        w = pw - 2 * mgap;
        h = (ph - tfsize - mgap) / n - mgap;
        for (var j = 0; j < n; i++) {
          rx = p1.x + mgap;
          ry = p1.y + tfsize + (h + mgap) * j + mgap;
          tx = p1.x + w * 0.5 + mgap;
          if (Array.isArray(node.children[i].children)) {
            ty = p1.y + tfsize + (h + mgap) * j + fontsize + mgap;
          } else {
            ty = p1.y + tfsize + (h + mgap) * j + h * 0.5 + mgap;
          }

          g = graph
            .append("g")
            .attr("class", "node")
            .attr("id", "g_" + node.children[i].guid);
          g.append("rect")
            .attr("x", rx)
            .attr("y", ry)
            .attr("ry", 20-deep*4)
            .attr("ry", 20-deep*4)
            .attr("width", w)
            .attr("height", h)
            .attr("stroke", "black")
            .attr("fill", color)
            .attr("stroke-width", strokewidth);
          /***********customize-start***************/
          if (node.isLast) {
            g.append("circle")
            .attr("cx", tx)
            .attr("cy", ty+12)
            .attr("r", 10)
            .attr("stroke", "blue")
            .attr("fill", color)
            .attr("stroke-width", strokewidth);
            g.append("text")
            .attr("x", tx)
            .attr("y", ty+16)
            .text('12%')
            .attr("style", "text-anchor:middle")
            .attr("font-size", fontsize);
          }
          /***********customize-end***************/
          g.append("text")
            .attr("x", tx)
            .attr("y", ty)
            .text(
              this.test(node,node.children[i])
              // node.children[i].data.code
              //   ? node.children[i].data.code
              //   : node.children[i].data.key_name
            )
            .attr("style", "text-anchor:middle")
            .attr("font-size", fontsize);
          if (Array.isArray(node.children[i].children)) {
            this.setChildren(
              node.children[i],
              { x: rx, y: ry },
              w,
              h,
              fontsize,
              deep + 1,
              idcName
            );
          }
        }
      }
    },
    dbClick (data) {
      console.log(data)
    },
    test(node, childrenData) {
        return childrenData.data.code
          ? childrenData.data.code
          : childrenData.data.key_name;
    },
    setEndpoint(node, p1, pw, ph, tfsize, deep, idcName) {
      let graph;
      if (idcName === "graphBig") {
        graph = d3.select("#graphBig").select("#g_" + node.guid);
      } else {
        graph = d3.select("#graph").select("#g_" + node.guid);
      }
      let n = node.children.length;
      let w, h, mgap, fontsize, strokewidth;
      let rx, ry, tx, ty, g;
      let color = colors[deep];
      if (pw > ph * 1.2) {
        mgap = (pw / n) * 0.04;
        fontsize =
          tfsize * 0.8 > (pw / n) * 0.1 ? (pw / n) * 0.1 : tfsize * 0.8;
        strokewidth = (pw / n) * 0.005;
        // w = (pw - mgap) / n - mgap;
        // h = ph - tfsize - 2 * mgap;
        w = 50;
        h = 20;
        for (var i = 0; i < n; i++) {
          rx = p1.x + (w + mgap) * i + mgap;
          ry = p1.y + tfsize + mgap;
          tx = p1.x + (w + mgap) * i + w * 0.5 + mgap;
          if (Array.isArray(node.children[i].children)) {
            ty = p1.y + tfsize + mgap + fontsize;
          } else {
            ty = p1.y + tfsize + mgap + h * 0.5;
          }

          g = graph
            .append("g")
            .attr("class", "node")
            .attr("id", "g_" + node.children[i].guid);
          g.append("rect")
            .attr("x", rx + 8)
            .attr("y", ry + 8)
            .attr("width", w)
            .attr("height", h)
            .attr("stroke", "black")
            .attr("fill", color)
            .attr("stroke-width", strokewidth);
          g.append("text")
            .attr("x", tx + 8)
            .attr("y", ty + 8)
            .text(
              node.children[i].data.code
                ? node.children[i].data.code
                : node.children[i].data.key_name
            )
            .attr("style", "text-anchor:middle")
            .attr("font-size", fontsize);
        }
      }
    },
    genDOT(idcData, linkData) {
      // let children = idcData.children || [];
      let links = linkData || [];
      let fsize = 16;
      let width = 16;
      let height = 12;
      let dots = [
        "digraph G {",
        "rankdir=TB nodesep=0.5;",
        `node [shape="box", fontsize="${fsize}", labelloc="t", penwidth="2"];`,
        `subgraph cluster_${idcData.data.guid} {`,
        `style="filled";color="${colors[0]}";`,
        `label="${idcData.data.name ||
          idcData.data.description ||
          idcData.data.code}";`,
        `size="${width},${height}";`,
        this.genChildren(idcData),
        this.genLink(links),
        "}}"
      ];
      return dots.join("");
    },
    genChildren(idcData) {
      const width = 16;
      const height = 12;
      let dots = [];
      const children = idcData.children || [];
      let layers = new Map();
      children.forEach(zone => {
        if (layers.has(zone.data.zone_layer.value)) {
          layers.get(zone.data.zone_layer.value).push(zone);
        } else {
          let layer = [];
          layer.push(zone);
          layers.set(zone.data.zone_layer.value, layer);
        }
      });
      if (layers.size) {
        layers.forEach(layer => {
          dots.push('{rank = "same";');
          let n = layers.size;
          let lg = (height - 3) / n;
          let ll = (width - 0.5 * layer.length) / layer.length;
          layer.forEach(zone => {
            let label;
            if (
              zone.data.code &&
              zone.data.code !== null &&
              zone.data.code !== ""
            ) {
              label = zone.data.code;
            } else {
              label = zone.data.key_name;
            }
            dots.push(
              `g_${zone.guid}[id="g_${zone.guid}", label="${label}", width=${ll},height=${lg}];`
            );
          });
          dots.push("}");
        });
      } else {
        dots.push(
          `g_${idcData.data.guid}[label=" ";color="${
            colors[0]
          }";width="${width - 0.5}";height="${height - 3}"]`
        );
      }
      return dots.join("");
    },
    genLink(links) {
      let result = "";
      links.forEach(link => {
        result += `${link.azone}->${link.bzone}[arrowhead="none"];`;
      });
      return result;
    }
  }
};
</script>

<style lang="less" scoped>
.resource-design-select-row {
  margin-bottom: 10px;
  display: flex;
  align-items: center;
}
.resource-design-tab-row {
  min-height: 50vh;
}
.graph-select {
  width: 400px;
}
.ivu-card-head p {
  height: 30px;
  line-height: 30px;
}
.filter-title {
  margin-right: 10px;
}
.graph-list {
  overflow-x: scroll;
  display: flex;
}
.graph-list > div {
  cursor: pointer;
}
.graph-container {
  width: 160px;
  height: 120px;
  float: left;
  margin-right: 5px;
  text-align: center;
}
.graph-container-big {
  margin-top: 20px;
}
</style>
