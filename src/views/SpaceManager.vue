<template>
  <div id='spacemanager'>
      <div class='controls'>
        Show files larger than:
      </div>
      <div class='canvas'>
      </div>
  </div>
</template>
<script>
import { mapState } from 'vuex'
import * as d3 from 'd3'
import store from '@/store'

export default {
  name: 'spacemanager',
  computed: mapState([ 'user' ]),
  mounted () {
    var loadVisualization = function (minSize) {
      var width = 960
      var height = 500

      var x = d3.scale.linear()
        .range([0, width])

      var y = d3.scale.linear()
        .range([0, height])

      var color = d3.scale.category20c()

      var partition = d3.layout.partition()
        .value(function (d) { return d.size })

      var canvas = d3.select('#spacemanager .canvas')
      canvas.select('svg').remove()
      var svg = canvas.append('svg')
        .attr('width', '100%')
        .attr('preserveAspectRatio', 'xMinYMin meet')
        .attr('viewBox', '0 0 ' + width + ' ' + height)

      var rect = svg.selectAll('rect')
      var textNames
      var textSizes
      var root = '/'
      d3.json(`${store.state.baseURL}/api/space/get?path=${root}`, function (error, data) {
        if (error) {
          console.error('An error has occured: ' + error)
        }

        var tree = {}
        var items = data.items
        for (var i = 0; i < items.length; i++) {
          if (!items[i].isDir && items[i].size < minSize) continue
          var path = items[i].virtualPath
          var size = items[i].size
          var parent = items[i].virtualPath.substring(0, items[i].virtualPath.length - items[i].name.length - 1)
          if (parent.length === 0) parent = '/'
          if (!(path in tree)) {
            tree[path] = {}
            tree[path].children = []
            tree[path].totalSize = 0
          }
          tree[path].parent = parent
          tree[path].name = path
          tree[path].size = size
          if (!items[i].isDir) {
            tree[path].totalSize += size
          }

          if (!(parent in tree)) {
            tree[parent] = {}
            tree[parent].children = []
            tree[parent].name = parent
            tree[parent].size = 0
            tree[parent].totalSize = 0
            tree[parent].parent = null
          }
          tree[parent].children.push(tree[path])

          while (parent != null) {
            tree[parent].totalSize += tree[path].totalSize
            parent = tree[parent].parent
          }
        }

        var nodeEnter = rect
          .data(partition.nodes(tree[root]))
          .enter()

        rect = nodeEnter.append('rect')
          .attr('x', function (d) { return x(d.x) })
          .attr('y', function (d) { return y(d.y) })
          .attr('width', function (d) { return x(d.dx) })
          .attr('height', function (d) { return y(d.dy) })
          .attr('fill', function (d) { return color(d.name) })
          .on('click', clicked)

        textNames = nodeEnter.append('text')
          .attr('x', function (d) { return x(d.x) + x(d.dx) / 2 })
          .attr('y', function (d) { return y(d.y) + y(d.dy) / 2 })
          .attr('class', 'text')
          .attr('text-anchor', 'middle')
          .text(function (d) { return truncate(pathBase(d.name), x(d.dx), 7) })

        textSizes = nodeEnter.append('text')
          .attr('x', function (d) { return x(d.x) + x(d.dx) / 2 })
          .attr('y', function (d) { return 12 + y(d.y) + y(d.dy) / 2 })
          .attr('class', 'text')
          .attr('text-anchor', 'middle')
          .text(function (d) { return truncate(formatSize(d.totalSize), x(d.dx), 7) })
      })

      function clicked (d) {
        x.domain([d.x, d.x + d.dx])
        y.domain([d.y, 1]).range([d.y ? 20 : 0, height])

        rect.transition()
          .duration(750)
          .attr('x', function (d) { return x(d.x) })
          .attr('y', function (d) { return y(d.y) })
          .attr('width', function (d) { return x(d.x + d.dx) - x(d.x) })
          .attr('height', function (d) { return y(d.y + d.dy) - y(d.y) })

        textNames.transition()
          .duration(750)
          .attr('x', function (d) { return x(d.x) + (x(d.x + d.dx) - x(d.x)) / 2 })
          .attr('y', function (d) { return y(d.y) + (y(d.y + d.dy) - y(d.y)) / 2 })
          .text(function (d) { return truncate(pathBase(d.name), x(d.x + d.dx) - x(d.x), 7) })

        textSizes.transition()
          .duration(750)
          .attr('x', function (d) { return x(d.x) + (x(d.x + d.dx) - x(d.x)) / 2 })
          .attr('y', function (d) { return 12 + y(d.y) + (y(d.y + d.dy) - y(d.y)) / 2 })
          .text(function (d) { return truncate(formatSize(d.totalSize), x(d.x + d.dx) - x(d.x), 7) })
      }

      function truncate (text, width, pxPerChar) {
        var maxChars = width / pxPerChar
        if (text.length <= maxChars) {
          return text
        }

        if (maxChars < 4) {
          return ''
        }

        return text.substring(0, maxChars - 3) + '...'
      }

      function pathBase (path) {
        if (path.length === 1 && path[0] === '/') {
          return path
        }
        var baseReversed = ''
        for (var i = path.length - 1; i >= 0; i--) {
          if (path[i] === '/') break
          baseReversed += path[i]
        }

        var base = ''
        for (var j = baseReversed.length - 1; j >= 0; j--) {
          base += baseReversed[j]
        }

        return base
      }

      function formatSize (bytes) {
        var sizes = [ 'B', 'KB', 'MB', 'GB', 'TB' ]
        var pos = 0
        while (bytes >= 1000 && pos < sizes.length - 1) {
          bytes /= 1000
          pos++
        }
        var bytesFormatted = Math.round(bytes * 100) / 100
        return bytesFormatted + ' ' + sizes[pos]
      }
    }

    var thresholds = [
      { key: 'All sizes', value: 0 },
      { key: '1 MB', value: 1000 * 1000 },
      { key: '10 MB', value: 10 * 1000 * 1000 },
      { key: '100 MB', value: 100 * 1000 * 1000 },
      { key: '1 GB', value: 1 * 1000 * 1000 * 1000 },
      { key: '10 GB', value: 10 * 1000 * 1000 * 1000 },
      { key: '20 GB', value: 20 * 1000 * 1000 * 1000 },
      { key: '50 GB', value: 50 * 1000 * 1000 * 1000 },
      { key: '100 GB', value: 100 * 1000 * 1000 * 1000 }
    ]

    d3.select('.controls')
      .append('div')
      .selectAll('span')
      .data(thresholds)
      .enter()
      .append('span')
      .text(function (d) { return d.key })
      .on('click', function (d) {
        loadVisualization(d.value)
      })

    loadVisualization(10 * 1000 * 1000)
  }
}
</script>