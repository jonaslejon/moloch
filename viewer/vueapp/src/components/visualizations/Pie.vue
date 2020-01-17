<template>
  <div>
    <div id="my_dataviz"
      class="horizontal-center mt-4">
    </div>
  </div>
</template>

<script>
import * as d3 from 'd3';
import 'd3-interpolate';

let width = window.innerWidth;
let height = 600;
let radius = Math.min(width, height) / 2;
let g, svg, pie, arc, outerArc;

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

function polylineTransition (d) {
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

export default {
  name: 'MolochPie',
  props: {
    graphData: Array
  },
  watch: {
    'graphData': function (newVal, oldVal) {
      this.applyGraphData(this.formatData(newVal));
    }
  },
  mounted: function () {
    this.initializeGraph(this.formatData(this.graphData));
  },
  methods: {
    /**
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
     * (key = data name, count = data value, idx = index to be added to the pie)
     * @param {Array} data The data array the format
     * @returns {Object} formattedData The formatted data object
     */
    formatData: function (data) {
      this.shuffle(data); // shuffle so the labels don't collide as easily
      let formattedData = {};
      for (let item of data) {
        formattedData[item.name] = { value: item.count, idx: item.idx };
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
      svg = d3.select('#my_dataviz')
        .append('svg')
        .attr('width', width)
        .attr('height', height)
        .append('g')
        .attr('transform', 'translate(' + width / 2 + ',' + height / 2 + ')');

      g = svg.append('g');

      g.attr('class', 'slices');
      g.attr('class', 'labels');
      g.attr('class', 'lines');

      pie = d3.pie().value((d) => {
        if (d && d.value && d.value.value) {
          return d.value.value;
        };
      }).sort((a, b) => {
        if (a && a.value && a.value.idx) {
          return a.value.idx - b.value.idx;
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
        .attr('dy', '.25rem')
        .text(getLabelText);

      // apply text location and transition to new labels
      g.datum(d3.entries(data))
        .selectAll('text')
        .data(pie(d3.entries(d3)))
        .transition().duration(1000)
        .attrTween('transform', textTransform)
        .styleTween('text-anchor', textTransition);

      // remove any text not being used
      g.datum(d3.entries(data))
        .selectAll('text')
        .data(pie(d3.entries(data)))
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
        .attrTween('points', polylineTransition);

      // remove any lines not being used
      g.datum(d3.entries(data))
        .selectAll('polyline')
        .data(pie(d3.entries(data)))
        .exit().remove();
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
