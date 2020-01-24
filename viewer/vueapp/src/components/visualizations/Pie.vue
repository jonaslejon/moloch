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
            query-param="field"
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
let width = window.innerWidth;
let height = 600;
let radius = Math.min(width, height) / 2;
let g, g2, svg, pie, arc, arc2, outerArc, outerArc2;

// pie functions
function midAngle (d) {
  return d.startAngle + (d.endAngle - d.startAngle) / 2;
}

function sliceTransition (d) {
  this._current = this._current || d;
  let interpolate = d3.interpolate(this._current, d);
  this._current = interpolate(0);
  return function (t) {
    return arc(interpolate(t));
  };
};

// TODO
function sliceTransition2 (d) {
  this._current = this._current || d;
  let interpolate = d3.interpolate(this._current, d);
  this._current = interpolate(0);
  return function (t) {
    return arc2(interpolate(t));
  };
}

function textTransform (d) {
  this._current = this._current || d;
  const interpolate = d3.interpolate(this._current, d);
  this._current = interpolate(0);
  return function (t) {
    const d2 = interpolate(t);
    let pos = outerArc.centroid(d2);
    pos[0] = radius * (midAngle(d2) < Math.PI ? 1 : -1);
    return 'translate(' + pos + ')';
  };
};

// TODO
function textTransform2 (d) {
  this._current = this._current || d;
  const interpolate = d3.interpolate(this._current, d);
  this._current = interpolate(0);
  return function (t) {
    const d2 = interpolate(t);
    let pos = outerArc2.centroid(d2);
    pos[0] = radius * (midAngle(d2) < Math.PI ? 1 : -1);
    return 'translate(' + pos + ')';
  };
};

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
  return `${d.data.key} (${d.value})`;
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

// TODO
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
      this.applyGraphData(this.formatData(newVal));
      // if there is more data to fetch than the spigraph component can provide,
      // fetch it from the spigraphpie endpoint
      if (this.fieldTypeaheadList.length) {
        this.loadData();
      }
    }
  },
  mounted: function () {
    this.initializeGraph(this.formatData(this.graphData));
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
      // TODO add back the labels and lines for the inner pie
      this.loadData(); // TODO need this?
    },
    /* event functions ----------------------------------------------------- */
    changeField: function (field) {
      // TODO only allow 2 deep?
      this.fieldTypeaheadList.push(field);
      this.loadData();
    },
    /* helper functions ---------------------------------------------------- */
    /**
     * TODO remove if we don't shuffle the data
     * Add a random index to each piece of data to randomize the rendering
     * in the pi graph so that labels don't collide as easily
     * modifies the array itself, doesn't return anything
     * @param {Array} data The data to add indices
     */
    shuffle: function (data) {
      let numArr = [];
      for (let i = 0; i < data.length; i++) {
        numArr[i] = i;
      }
      numArr.sort((a, b) => {
        return Math.floor(Math.random() * 3) - 1;
      });
      for (let i = 0, len = data.length; i < len; i++) {
        data[i].idx = numArr[i];
      }
    },
    /**
     * Turn the data array into an object and only preserve neceessary info
     * This is only needed when data is coming from the spigraph loadData func
     * (key = data name, count = data value, idx = index to be added to the pie)
     * @param {Array} data The data array the format
     * @returns {Object} formattedData The formatted data object
     */
    formatData: function (data) {
      // TODO remove if we want data in descending order
      // this.shuffle(data); // shuffle so the labels don't collide as easily
      let formattedData = {};
      for (let item of data) {
        // TODO remove if we don't shuffle the data
        // formattedData[item.name] = { value: item.count, idx: item.idx };
        formattedData[item.name] = item.count;
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
     * Initializes the graph by adding the svg to the page, setting the value
     * of the pie slices, sorting the pie slices by index, and defining the
     * inner and outer arc geometry
     * @params {Object} data The data to construct the pie
     */
    initializeGraph: function (data) {
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

      arc = d3.arc()
        .innerRadius(radius * 0.7)
        .outerRadius(radius * 0.4)
        .cornerRadius(6);

      outerArc = d3.arc()
        .innerRadius(radius * 0.9)
        .outerRadius(radius * 0.9);

      this.applyGraphData(data);
    },
    /**
     * Applies the graph data to the pie chart by adding the slices, text labels,
     * and lines from the slices to the text labels
     * It also adds the colors and transitions to the pie graph
     * (works for new a new pie as well as updating the pie)
     */
    applyGraphData: function (data) {
      let colors = this.generateColors(data);

      // PIE SLICES -------------------------- //
      // add any new slices
      g.datum(d3.entries(data))
        .selectAll('path')
        .data(pie(d3.entries(data)))
        .enter()
        .append('path')
        .attr('d', arc)
        .attr('stroke', 'white')
        .style('stroke-width', '2px')
        .style('opacity', 0.8);

      // apply color to ALL of the slices (not just the new ones)
      g.selectAll('path')
        .attr('fill', (d) => {
          return colors(d.data.key);
        });

      // add transition to new slices
      g.datum(d3.entries(data))
        .selectAll('path')
        .data(pie(d3.entries(data)))
        .transition().duration(1000)
        .attrTween('d', sliceTransition);

      // remove any slices not being used
      g.datum(d3.entries(data))
        .selectAll('path')
        .data(pie(d3.entries(data)))
        .exit().remove();

      // TEXT LABELS ------------------------- //
      // add any new text
      g.datum(d3.entries(data))
        .selectAll('text')
        .data(pie(d3.entries(data)))
        .enter()
        .append('text')
        .attr('dy', '.25rem');

      // apply text location and transition to new labels
      g.datum(d3.entries(data))
        .selectAll('text')
        .data(pie(d3.entries(data)))
        .transition().duration(1000)
        .attrTween('transform', textTransform)
        .styleTween('text-anchor', textTransition);

      // remove any text not being used and apply text label data
      g.datum(d3.entries(data))
        .selectAll('text')
        .data(pie(d3.entries(data)))
        .text(getLabelText)
        .exit().remove();

      // LINE TO LABEL ----------------------- //
      // add any new lines from slices to labels
      g.datum(d3.entries(data))
        .selectAll('polyline')
        .data(pie(d3.entries(data)))
        .enter()
        .append('polyline');

      // apply transitions to new lines
      g.datum(d3.entries(data))
        .selectAll('polyline')
        .transition().duration(1000)
        .attrTween('points', polylineTransform);

      // remove any lines not being used
      g.datum(d3.entries(data))
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

        // TODO need to redraw the entire graph because the inner shit comes
        // out different

        // add the data to the graph
        // TODO maybe determine if there is multilevel data
        // (if we need 2 different functions, one for single level data one for multilevel data)
        let innerData = {};
        let outerData = {};
        let index = 0;
        let parentIndex = 0;

        // create a data object and identify the parent index (which bucket a
        // value belongs to) and the index of the value itself (for coloring)
        for (let item in response.data) {
          innerData[item] = response.data[item].value;
          let subIndex = 0;
          for (let subItem in response.data[item].subData) {
            // TODO do we want unique colors or colors per key/name?
            outerData[`${subItem}-${parentIndex}`] = {
            // outerData[subItem] = {
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

        this.applyGraphData(innerData);
        this.applyMoreGraphData(outerData);
      }).catch((error) => {
        pendingPromise = null;
        this.$emit('toggleLoad', false);
        this.$emit('toggleError', error.text || error);
      });
    },
    // TODO make this apply more graph data (2nd and 3rd layers)???????????????
    // TODO generalize this function and combine it with applyGraphData????????
    applyMoreGraphData: function (data) {
      let colors = this.generateColors(data);

      // create new arc functions to put data outisde of current data
      arc2 = d3.arc()
        .innerRadius(radius * 0.9)
        .outerRadius(radius * 0.7)
        .cornerRadius(6);

      outerArc2 = d3.arc()
        .innerRadius(radius * 0.95)
        .outerRadius(radius * 0.95);

      // TODO ECR REMOVE
      // console.log('adding data to graph:', data);
      // console.log('d3 formatted data', d3.entries(data));

      // TODO remove labels from inner pie?
      // remove labels from inner pie
      g.datum(d3.entries({}))
        .selectAll('text')
        .data(pie(d3.entries({})))
        .exit().remove();
      // remove lines from slices to labels from inner pie
      g.datum(d3.entries({}))
        .selectAll('polyline')
        .data(pie(d3.entries({})))
        .exit().remove();

      // add another g to add the new pie data
      g2 = svg.append('g');

      // PIE SLICES -------------------------- //
      // add any new slices
      g2.datum(d3.entries(data))
        .selectAll('path')
        .data(pie(d3.entries(data)))
        .enter()
        .append('path')
        .attr('d', arc2)
        .attr('stroke', 'white')
        .style('stroke-width', '2px')
        .style('opacity', 0.8);

      // apply color to the new slices
      g2.selectAll('path')
        .attr('fill', (d) => {
          // TODO do we want unique colors or colors per key/name?
          // return colors(d.data.key);
          return colors(`${d.data.key}-${d.data.parentIndex}`);
        });

      // add transition to new slices
      g2.datum(d3.entries(data))
        .selectAll('path')
        .data(pie(d3.entries(data)))
        .transition().duration(1000)
        .attrTween('d', sliceTransition2);

      // remove any slices not being used
      g2.datum(d3.entries(data))
        .selectAll('path')
        .data(pie(d3.entries(data)))
        .exit().remove();

      // TODO hover over item to see values?
      // TODO or just show largest sections?
      // TODO or both!?
      // TEXT LABELS ------------------------- //
      // add any new text
      g2.datum(d3.entries(data))
        .selectAll('text')
        .data(pie(d3.entries(data)))
        .enter()
        .append('text')
        .attr('dy', '.25rem');

      // apply text location and transition to new labels
      g2.datum(d3.entries(data))
        .selectAll('text')
        .data(pie(d3.entries(data)))
        .transition().duration(1000)
        .attrTween('transform', textTransform2)
        .styleTween('text-anchor', textTransition);

      // remove any text not being used and apply text label data
      g2.datum(d3.entries(data))
        .selectAll('text')
        .data(pie(d3.entries(data)))
        .text((d) => {
          // only return the top 2 labels
          if (d.data.value.subIndex < 2) {
            return `${d.data.value.name} (${d.data.value.value})`;
          }
        })
        .exit().remove();

      // LINE TO LABEL ----------------------- //
      // add any new lines from slices to labels
      g2.datum(d3.entries(data))
        .selectAll('polyline')
        .data(pie(d3.entries(data)))
        .enter()
        .append('polyline');

      // apply transitions to new lines
      g2.datum(d3.entries(data))
        .selectAll('polyline')
        .transition().duration(1000)
        .attrTween('points', polylineTransform2);

      // remove any lines not being used
      g2.datum(d3.entries(data))
        .selectAll('polyline')
        .data(pie(d3.entries(data)))
        .exit().remove();
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
