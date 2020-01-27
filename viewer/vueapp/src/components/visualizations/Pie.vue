<template>
  <div>

    <!-- field select -->
    <div class="form-inline pl-1">
      <div class="form-group"
        v-if="fields && fields.length">
        <div class="input-group input-group-sm mr-2">
          <span class="input-group-prepend cursor-help"
            v-b-tooltip.hover
            title="SPI Graph Field">
            <span class="input-group-text">
              Add another field:
            </span>
          </span>
          <moloch-field-typeahead
            :fields="fields"
            @fieldSelected="changeField"
            page="SpigraphSubfield">
          </moloch-field-typeahead>
        </div>
      </div>
      <template v-for="(field, index) in fieldTypeaheadList">
        <label class="badge badge-secondary mr-1 mb-1 help-cursor"
          :title="field.help"
          v-b-tooltip.hover
          v-if="field"
          :key="`${field.dbField}-${index}`">
          {{ field.friendlyName }}
          <button class="close ml-2"
            style="margin-top:-8px"
            @click="removeField(index)">
            <span>&times;</span>
          </button>
        </label>
      </template>
    </div> <!-- /field select -->

    <!-- pie chart area -->
    <div id="pie-area"
      class="horizontal-center">
    </div>
    <!-- /pie chart area -->

  </div>
</template>

<script>
// import external
import Vue from 'vue';
import * as d3 from 'd3';
import 'd3-interpolate';
// import services
import SpigraphService from '../spigraph/SpigraphService';
// import internal
import MolochFieldTypeahead from '../utils/FieldTypeahead';
// import utils
import Utils from '../utils/utils';

let pendingPromise; // save a pending promise to be able to cancel it

// page pie variables
let height = 600;
let width = window.innerWidth;
let radius = Math.min(width, height) / 2;
let g, g2, svg, pie, tooltip;

const arc = d3.arc()
  .innerRadius(radius * 0.7)
  .outerRadius(radius * 0.4)
  .cornerRadius(6);

const outerArc = d3.arc()
  .innerRadius(radius * 0.9)
  .outerRadius(radius * 0.9);

const arc2 = d3.arc()
  .innerRadius(radius * 0.9)
  .outerRadius(radius * 0.7)
  .cornerRadius(6);

const outerArc2 = d3.arc()
  .innerRadius(radius * 0.95)
  .outerRadius(radius * 0.95);

const innerLabelArc = d3.arc()
  .innerRadius(radius * 0.7 - 25)
  .outerRadius(radius * 0.4 - 25);

// pie functions
function midAngle (d) {
  return d.startAngle + (d.endAngle - d.startAngle) / 2;
}

function sliceTransition (d, arcFunc, _current) {
  let interpolate = d3.interpolate(_current, d);
  _current = interpolate(0);
  return function (t) {
    return arcFunc(interpolate(t));
  };
};

// put the text outside of the pie slices
function textTransform (d, arcFunc, _current) {
  const interpolate = d3.interpolate(_current, d);
  _current = interpolate(0);
  return function (t) {
    const d2 = interpolate(t);
    let pos = arcFunc.centroid(d2);
    pos[0] = radius * (midAngle(d2) < Math.PI ? 1 : -1);
    return `translate(${pos})`;
  };
};

// put the text on top of the pie slices
function textTransformOnSlice (d) {
  return `translate(${innerLabelArc.centroid(d)})`;
}

function textTransition (d) {
  this._current = this._current || d;
  const interpolate = d3.interpolate(this._current, d);
  this._current = interpolate(0);
  return function (t) {
    var d2 = interpolate(t);
    return midAngle(d2) < Math.PI ? 'start' : 'end';
  };
}

function getLabelText (d) {
  return `${d.data.value.name} (${d.data.value.value})`;
}

function getTopLabelText (d) {
  if (d.data.value.subIndex < 2) {
    return `${d.data.value.name} (${d.data.value.value})`;
  }
}

function polylineTransform (d) {
  this._current = this._current || d;
  const interpolate = d3.interpolate(this._current, d);
  this._current = interpolate(0);
  return function (t) {
    const d2 = interpolate(t);
    let pos = outerArc.centroid(d2);
    pos[0] = radius * 0.95 * (midAngle(d2) < Math.PI ? 1 : -1);
    let arcCentroid = arc.centroid(d2);
    arcCentroid[0] = arcCentroid[0] - (arcCentroid[0] * -0.2);
    arcCentroid[1] = arcCentroid[1] - (arcCentroid[1] * -0.2);
    return [arcCentroid, outerArc.centroid(d2), pos];
  };
}

function polylineTransform2 (d) {
  if (d.data.value.subIndex > 1) {
    return; // only display the first 2
  }
  this._current = this._current || d;
  const interpolate = d3.interpolate(this._current, d);
  this._current = interpolate(0);
  return function (t) {
    const d2 = interpolate(t);
    let pos = outerArc2.centroid(d2);
    pos[0] = radius * 0.95 * (midAngle(d2) < Math.PI ? 1 : -1);
    let arcCentroid = arc2.centroid(d2);
    arcCentroid[0] = arcCentroid[0] - (arcCentroid[0] * -0.1);
    arcCentroid[1] = arcCentroid[1] - (arcCentroid[1] * -0.1);
    return [arcCentroid, outerArc2.centroid(d2), pos];
  };
}

function mouseover (d) {
  tooltip.style('opacity', 1)
    .html(() => {
      if (d.data.value.name) {
        return `${d.data.value.name} <br><strong>${d.data.value.value}</strong>`;
      } else {
        return `${d.data.key}: <br><strong>${d.data.value}</strong>`;
      }
    });
  d3.select(this).style('opacity', 1);
};

function mouseleave (d) {
  tooltip.style('opacity', 0);
  d3.select(this).style('opacity', 0.8);
};

export default {
  name: 'MolochPie',
  components: { MolochFieldTypeahead },
  props: {
    baseField: String,
    graphData: Array,
    fields: Array,
    query: Object
  },
  data: function () {
    return {
      fieldTypeaheadList: []
    };
  },
  watch: {
    'graphData': function (newVal, oldVal) {
      // if there is more data to fetch than the spigraph component can provide,
      // fetch it from the spigraphpie endpoint
      if (this.fieldTypeaheadList.length) {
        this.loadData();
      } else {
        let data = this.formatDataFromSpigraph(newVal);
        this.applyGraphData(data, g, arc, outerArc, polylineTransform, getLabelText);
      }
    }
  },
  mounted: function () {
    this.initializeGraph(this.formatDataFromSpigraph(this.graphData));
  },
  methods: {
    /* exposed page functions ---------------------------------------------- */
    /**
     * Removes a field from the field typeahead list and loads the data
     * @param {Number} index The index of the field typeahead list to remove
     */
    removeField: function (index) {
      this.fieldTypeaheadList.splice(index, 1);

      if (g2) {
        // remove slices
        g2.datum(d3.entries({}))
          .selectAll('path')
          .data(pie(d3.entries({})))
          .exit().remove();
        // remove labels
        g2.datum(d3.entries({}))
          .selectAll('text')
          .data(pie(d3.entries({})))
          .exit().remove();
        // remove polylines
        g2.datum(d3.entries({}))
          .selectAll('polyline')
          .data(pie(d3.entries({})))
          .exit().remove();
      }

      this.loadData();
    },
    /* event functions ----------------------------------------------------- */
    changeField: function (field) {
      // TODO allow 2 items in this array?
      if (this.fieldTypeaheadList.length > 0) {
        this.$set(this.fieldTypeaheadList, 0, field);
      } else {
        this.fieldTypeaheadList.push(field);
      }

      this.loadData();
    },
    /* helper functions ---------------------------------------------------- */
    /**
     * Turn the data array into an object and only preserve neceessary info
     * This is only needed when data is coming from the spigraph loadData func
     * (key = data name, count = data value, idx = index to be added to the pie)
     * @param {Array} data The data array the format
     * @returns {Object} formattedData The formatted data object
     */
    formatDataFromSpigraph: function (data) {
      let formattedData = {};
      for (let item of data) {
        formattedData[item.name] = {
          name: item.name,
          value: item.count
        };
      }
      return formattedData;
    },
    /**
     * Generates a list of colors (RAINBOW) based on the length of the data
     * @param {Object} data The data object to calculate colors for
     * @returns {Function} colors Function to retrieve a color per data point
     */
    generateColors: function (data) {
      const colorScale = d3.interpolateHslLong('red', 'purple');
      const dataLength = Object.keys(data).length;
      const intervalSize = 1 / dataLength;

      let colorArray = [];
      for (let i = 0; i < dataLength; i++) {
        let color = colorScale(i * intervalSize);
        colorArray.push(color);
      }

      // set the color scale
      return d3.scaleOrdinal()
        .domain(data)
        .range(colorArray);
    },
    /**
     * Initializes the graph by adding the svg to the page once the component
     * has mounted and the pie area container is present.
     * Sets the value of the pie slices and sorts the pie slices by index
     * Applies the initial data to the pie graph
     * @param {Object} data The data to construct the pie
     */
    initializeGraph: function (data) {
      // PIE SETUP --------------------------- //
      svg = d3.select('#pie-area')
        .append('svg')
        .attr('width', width)
        .attr('height', height)
        .append('g')
        .attr('transform', 'translate(' + width / 2 + ',' + height / 2 + ')');

      g = svg.append('g');

      pie = d3.pie().value((d) => {
        if (d && d.value && d.value.value) {
          return d.value.value;
        };
        return d.value;
      }).sort((a, b) => {
        if (a && a.value && a.value.index) {
          return a.value.index - b.value.index;
        }
        return true;
      });

      // TOOLTIPS ---------------------------- //
      tooltip = d3.select('#pie-area')
        .append('div')
        .style('opacity', 0)
        .attr('class', 'tooltip')
        .style('text-align', 'center')
        .style('padding', '5px');

      this.applyGraphData(data, g, arc, outerArc, polylineTransform, getLabelText);
    },
    /**
     * Applies the graph data to the pie chart by adding the slices, text labels,
     * and lines from the slices to the text labels
     * It also adds the colors and transitions to the pie graph
     * (works for new a new pie as well as updating the pie)
     * @param {Object} data             The data to add to the graph
     * @param {Object} gArea            The d3 g element for the data
     * @param {Function} arcFunc        The arc function to apply to the slices
     * @param {Function} outerArcFunc   The outer arc function to apply to the slices
     * @param {Function} lineTransFunc  The function to apply to draw the lines
     * @param {Function} labelTextFunc  The function to show the text labels
     */
    applyGraphData: function (data, gArea, arcFunc, outerArcFunc, lineTransFunc, labelTextFunc) {
      let colors = this.generateColors(data);

      // PIE SLICES -------------------------- //
      // add any new slices
      gArea.datum(d3.entries(data))
        .selectAll('path')
        .data(pie(d3.entries(data)))
        .enter()
        .append('path')
        .attr('d', arc2)
        .attr('stroke', 'white')
        .style('stroke-width', '2px')
        .style('opacity', 0.8)
        .on('mouseover', mouseover)
        .on('mouseleave', mouseleave);

      // apply color to ALL of the slices (not just the new ones)
      gArea.selectAll('path')
        .attr('fill', (d) => {
          return colors(d.data.key);
        });

      // add transition to new slices
      gArea.datum(d3.entries(data))
        .selectAll('path')
        .data(pie(d3.entries(data)))
        .transition().duration(1000)
        .attrTween('d', (d) => {
          return sliceTransition(d, arcFunc, this._current || d);
        });

      // remove any slices not being used
      gArea.datum(d3.entries(data))
        .selectAll('path')
        .data(pie(d3.entries(data)))
        .exit().remove();

      // TEXT LABELS ------------------------- //
      // add any new text
      gArea.datum(d3.entries(data))
        .selectAll('text')
        .data(pie(d3.entries(data)))
        .enter()
        .append('text')
        .attr('dy', '.15rem');

      // apply text location and transition to new labels
      gArea.datum(d3.entries(data))
        .selectAll('text')
        .data(pie(d3.entries(data)))
        .transition().duration(1000)
        .attrTween('transform', (d) => {
          return textTransform(d, outerArcFunc, this._current || d);
        })
        .styleTween('text-anchor', textTransition);

      // remove any text not being used and apply text label data
      gArea.datum(d3.entries(data))
        .selectAll('text')
        .data(pie(d3.entries(data)))
        .text(labelTextFunc)
        .exit().remove();

      // LINE TO LABEL ----------------------- //
      // add any new lines from slices to labels
      gArea.datum(d3.entries(data))
        .selectAll('polyline')
        .data(pie(d3.entries(data)))
        .enter()
        .append('polyline');

      // apply transitions to new lines
      gArea.datum(d3.entries(data))
        .selectAll('polyline')
        .transition().duration(1000)
        .attrTween('points', lineTransFunc);

      // remove any lines not being used
      gArea.datum(d3.entries(data))
        .selectAll('polyline')
        .data(pie(d3.entries(data)))
        .exit().remove();
    },
    loadData: function () {
      this.$emit('toggleLoad', true);
      this.$emit('toggleError', '');

      // create unique cancel id to make canel req for corresponding es task
      const cancelId = Utils.createRandomString();
      this.query.cancelId = cancelId;

      const source = Vue.axios.CancelToken.source();
      const cancellablePromise = SpigraphService.getPie(this.query, source.token);

      // setup the query params
      let params = this.query;
      let exps = [ this.baseField ];

      for (let field of this.fieldTypeaheadList) {
        exps.push(field.exp);
      }

      params.exp = exps.toString(',');

      // set pending promise info so it can be cancelled
      pendingPromise = { cancellablePromise, source, cancelId };

      cancellablePromise.then((response) => {
        pendingPromise = null;
        this.$emit('toggleLoad', false);

        // format the data for the pie graph
        let innerData = {};
        let outerData = {};
        let index = 0;
        let parentIndex = 0;

        // create a data object and identify the parent index (which bucket a
        // value belongs to) and the index of the value itself (for coloring)
        for (let item in response.data) {
          innerData[item] = {
            name: item,
            value: response.data[item].value
          };
          let subIndex = 0;
          for (let subItem in response.data[item].subData) {
            outerData[`${subItem}-${parentIndex}`] = {
              value: response.data[item].subData[subItem],
              parentIndex: parentIndex,
              index: index,
              name: subItem,
              subIndex: subIndex
            };
            index++;
            subIndex++;
          }
          parentIndex++;
        }

        this.applyGraphData(innerData, g, arc, outerArc, polylineTransform, getLabelText);
        if (index > 0) { // if there's outer data
          // put inner labels on pie slices
          g.selectAll('text')
            .transition().duration(1000)
            .style('text-anchor', 'middle')
            .attr('transform', textTransformOnSlice)
            .styleTween('text-anchor', textTransition);

          // remove lines from slices to labels from inner pie
          g.datum(d3.entries({}))
            .selectAll('polyline')
            .data(pie(d3.entries({})))
            .exit().remove();

          // add another g to add the new pie data
          if (!g2) { g2 = svg.append('g'); }
          this.applyGraphData(outerData, g2, arc2, outerArc2, polylineTransform2, getTopLabelText);
        }
      }).catch((error) => {
        pendingPromise = null;
        this.$emit('toggleLoad', false);
        this.$emit('toggleError', error.text || error);
      });
    }
  },
  beforeDestroy: function () {
    if (pendingPromise) {
      pendingPromise.source.cancel();
      pendingPromise = null;
    }
  }
};
</script>

<style>
/* styling for the lines connecting the labels to the slices
   make sure they are lines, not triangles (no fill) */
polyline {
  fill: none;
  opacity: .3;
  stroke: black;
  stroke-width: 2px;
}
</style>
