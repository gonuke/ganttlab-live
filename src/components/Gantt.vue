<template>
  <div id="Gantt">
    <p v-if="tasks.length == 0">No tasks out there...</p>
    <div v-if="tasks.length > 0" id="chart"></div>
  </div>
</template>

<script>
import * as d3 from 'd3'
import moment from 'moment-timezone'

export default {
  name: 'gantt',
  data () {
    return {
      //
    }
  },
  computed: {
    tasks: function () {
      return this.$store.state.tasks
    }
  },
  watch: {
    tasks: function (newTasks) {
      this.refreshChart()
    }
  },
  methods: {
    // thank you Florian Roscheck for this, you made an awesome work I only needed to tweak a little
    visavailChart: function (event) {
      // define chart layout
      var margin = {
        // top margin includes title and legend
        top: 70,

        // right margin should provide space for last horz. axis title
        right: 40,

        bottom: 20,

        // left margin should provide space for y axis titles
        left: 250
      }

      // height of horizontal data bars
      var dataHeight = 18

      // spacing between horizontal data bars
      var lineSpacing = 14

      // vertical space for heading
      var paddingTopHeading = -50

      // vertical overhang of vertical grid lines on bottom
      var paddingBottom = 10

      // space for y axis titles
      var paddingLeft = -250

      var width = window.innerWidth - margin.left - margin.right - 20

      // title of chart is drawn or not (default: yes)
      var drawTitle = 0

      // year ticks to be emphasized or not (default: yes)
      var emphasizeYearTicks = 1

      // define chart pagination
      // max. no. of datasets that is displayed, 0: all (default: all)
      var maxDisplayDatasets = 0

      // dataset that is displayed first in the current
      // display, chart will show datasets "curDisplayFirstDataset" to
      // "curDisplayFirstDataset+maxDisplayDatasets"
      var curDisplayFirstDataset = 0

      // global div for taskTooltip
      var div = d3.select('body').append('div')
          .attr('class', 'taskTooltip')
          .style('opacity', 0)

      var definedBlocks = null
      var isDateOnlyFormat = null

      function chart (selection) {
        selection.each(function drawGraph (dataset) {
          // check which subset of datasets have to be displayed
          var maxPages = 0
          var startSet
          var endSet
          if (maxDisplayDatasets !== 0) {
            startSet = curDisplayFirstDataset
            if (curDisplayFirstDataset + maxDisplayDatasets > dataset.length) {
              endSet = dataset.length
            } else {
              endSet = curDisplayFirstDataset + maxDisplayDatasets
            }
            maxPages = Math.ceil(dataset.length / maxDisplayDatasets)
          } else {
            startSet = 0
            endSet = dataset.length
          }

          // append data attribute in HTML for pagination interface
          selection.attr('data-max-pages', maxPages)

          var noOfDatasets = endSet - startSet
          var height = dataHeight * noOfDatasets + lineSpacing * noOfDatasets - 1

          // check how data is arranged
          if (definedBlocks === null) {
            definedBlocks = 0
            for (var i = 0; i < dataset.length; i++) {
              if (dataset[i].data[0].length === 3) {
                definedBlocks = 1
                break
              } else {
                if (definedBlocks) {
                  throw new Error('Detected different data formats in input data. Format can either be ' +
                      'continuous data format or time gap data format but not both.')
                }
              }
            }
          }

          // parse data text strings to JavaScript date stamps
          var parseDate = d3.time.format('%Y-%m-%d')
          var parseDateTime = d3.time.format('%Y-%m-%d %H:%M:%S')
          var parseDateRegEx = new RegExp(/^\d{4}-\d{2}-\d{2}$/)
          var parseDateTimeRegEx = new RegExp(/^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}$/)
          if (isDateOnlyFormat === null) {
            isDateOnlyFormat = true
          }
          dataset.forEach(function (d) {
            d.data.forEach(function (d1) {
              if (!(d1[0] instanceof Date)) {
                if (parseDateRegEx.test(d1[0])) {
                  // d1[0] is date without time data
                  d1[0] = parseDate.parse(d1[0])
                } else if (parseDateTimeRegEx.test(d1[0])) {
                  // d1[0] is date with time data
                  d1[0] = parseDateTime.parse(d1[0])
                  isDateOnlyFormat = false
                } else {
                  throw new Error('Date/time format not recognized. Pick between \'YYYY-MM-DD\' or ' +
                    '\'YYYY-MM-DD HH:MM:SS\'.')
                }

                if (!definedBlocks) {
                  d1[2] = d3.time.second.offset(d1[0], d.interval_s)
                } else {
                  if (parseDateRegEx.test(d1[2])) {
                    // d1[2] is date without time data
                    d1[2] = parseDate.parse(d1[2])
                  } else if (parseDateTimeRegEx.test(d1[2])) {
                    // d1[2] is date with time data
                    d1[2] = parseDateTime.parse(d1[2])
                  } else {
                    throw new Error('Date/time format not recognized. Pick between \'YYYY-MM-DD\' or ' +
                        '\'YYYY-MM-DD HH:MM:SS\'.')
                  }
                }
              }
            })
          })

          // cluster data by dates to form time blocks
          dataset.forEach(function (series, seriesI) {
            var tmpData = []
            var dataLength = series.data.length
            series.data.forEach(function (d, i) {
              if (i !== 0 && i < dataLength) {
                if (d[1] === tmpData[tmpData.length - 1][1]) {
                  // the value has not changed since the last date
                  if (definedBlocks) {
                    if (tmpData[tmpData.length - 1][2].getTime() === d[0].getTime()) {
                      // end of old and start of new block are the same
                      tmpData[tmpData.length - 1][2] = d[2]
                      tmpData[tmpData.length - 1][3] = 1
                    } else {
                      tmpData.push(d)
                    }
                  } else {
                    tmpData[tmpData.length - 1][2] = d[2]
                    tmpData[tmpData.length - 1][3] = 1
                  }
                } else {
                  // the value has changed since the last date
                  d[3] = 0
                  if (!definedBlocks) {
                    // extend last block until new block starts
                    tmpData[tmpData.length - 1][2] = d[0]
                  }
                  tmpData.push(d)
                }
              } else if (i === 0) {
                d[3] = 0
                tmpData.push(d)
              }
            })
            dataset[seriesI].disp_data = tmpData
          })

          // determine start and end dates among all nested datasets
          var startDate = 0
          var endDate = 0

          dataset.forEach(function (series, seriesI) {
            if (series.disp_data.length > 0) {
              if (startDate === 0) {
                startDate = series.disp_data[0][0]
                endDate = series.disp_data[series.disp_data.length - 1][2]
              } else {
                if (series.disp_data[0][0] < startDate) {
                  startDate = series.disp_data[0][0]
                }
                if (series.disp_data[series.disp_data.length - 1][2] > endDate) {
                  endDate = series.disp_data[series.disp_data.length - 1][2]
                }
              }
            }
          })

          // define scales
          var xScale = d3.time.scale()
              .domain([startDate, endDate])
              .range([0, width])
              .clamp(1)

          // define axes
          var xAxis = d3.svg.axis()
              .scale(xScale)
              .orient('top')

          // create SVG element
          var svg = d3.select(this).append('svg')
              .attr('width', width + margin.left + margin.right - 20)
              .attr('height', height + margin.top + margin.bottom)
              .append('g')
              .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')')

          // create basic element groups
          svg.append('g').attr('id', 'g_title')
          svg.append('g').attr('id', 'g_axis')
          svg.append('g').attr('id', 'g_data')

          // create y axis labels
          svg.select('#g_axis').selectAll('text')
              .data(dataset.slice(startSet, endSet))
              .enter()
              .append('a')
              .attr('xlink:href', function (d) {
                return d.link
              })
              .attr('xlink:show', 'new')
              .append('text')
              .attr('x', paddingLeft)
              .attr('y', lineSpacing + dataHeight / 2)
              .text(function (d) {
                return d.title
              })
              .attr('transform', function (d, i) {
                return 'translate(0,' + ((lineSpacing + dataHeight) * i) + ')'
              })
              .attr('class', 'ytitle')

          // create vertical grid
          svg.select('#g_axis').selectAll('line.vert_grid').data(xScale.ticks())
              .enter()
              .append('line')
              .attr({
                'class': 'vert_grid',
                'x1': function (d) {
                  return xScale(d)
                },
                'x2': function (d) {
                  return xScale(d)
                },
                'y1': 0,
                'y2': dataHeight * noOfDatasets + lineSpacing * noOfDatasets - 1 + paddingBottom
              })

          // create horizontal grid
          svg.select('#g_axis').selectAll('line.horz_grid').data(dataset)
              .enter()
              .append('line')
              .attr({
                'class': 'horz_grid',
                'x1': 0,
                'x2': width,
                'y1': function (d, i) {
                  return ((lineSpacing + dataHeight) * i) + lineSpacing + dataHeight / 2
                },
                'y2': function (d, i) {
                  return ((lineSpacing + dataHeight) * i) + lineSpacing + dataHeight / 2
                }
              })

          // create x axis
          svg.select('#g_axis').append('g')
              .attr('class', 'axis')
              .call(xAxis)

          // make y groups for different data series
          var g = svg.select('#g_data').selectAll('.g_data')
              .data(dataset.slice(startSet, endSet))
              .enter()
              .append('a')
              .attr('xlink:href', function (d) {
                return d.link
              })
              .attr('xlink:show', 'new')
              .append('g')
              .attr('transform', function (d, i) {
                return 'translate(0,' + ((lineSpacing + dataHeight) * i) + ')'
              })
              .attr('class', 'dataset')

          // add data series
          g.selectAll('rect')
              .data(function (d) {
                return d.disp_data
              })
              .enter()
              .append('rect')
              .attr('x', function (d) {
                return xScale(d[0])
              })
              .attr('y', lineSpacing)
              .attr('width', function (d) {
                return (xScale(d[2]) - xScale(d[0]))
              })
              .attr('height', dataHeight)
              .attr('class', function (d) {
                if (d[1] === 1) {
                  return 'rect_has_data'
                }
                return 'rect_has_no_data'
              })
              .on('mouseover', function (d, i) {
                var matrix = this.getScreenCTM().translate(+this.getAttribute('x'), +this.getAttribute('y'))
                div.transition()
                    .duration(200)
                    .style('opacity', 0.9)
                div.html(function () {
                  var output = ''
                  if (d[1] === 1) {
                    output = '<i class="fa fa-fw fa-check taskTooltip_has_data"></i>'
                  } else {
                    output = '<i class="fa fa-fw fa-times taskTooltip_has_no_data"></i>'
                  }
                  if (isDateOnlyFormat) {
                    if (d[2] > d3.time.second.offset(d[0], 86400)) {
                      output = output + moment(parseDate(d[0])).format('l') +
                          ' - ' + moment(parseDate(d[2])).format('l')
                    } else {
                      output = output + moment(parseDate(d[0])).format('l')
                    }
                  } else {
                    if (d[2] > d3.time.second.offset(d[0], 86400)) {
                      output = output + moment(parseDateTime(d[0])).format('l') + ' ' +
                          moment(parseDateTime(d[0])).format('LTS') + ' - ' +
                          moment(parseDateTime(d[2])).format('l') + ' ' +
                          moment(parseDateTime(d[2])).format('LTS')
                    } else {
                      output = output + moment(parseDateTime(d[0])).format('LTS') + ' - ' +
                        moment(parseDateTime(d[2])).format('LTS')
                    }
                  }
                  return output
                })
                .style('left', function () {
                  return window.pageXOffset + matrix.e + 'px'
                })
                .style('top', function () {
                  return window.pageYOffset + matrix.f - 11 + 'px'
                })
                .style('height', dataHeight + 11 + 'px')
              })
              .on('mouseout', function () {
                div.transition()
                    .duration(500)
                    .style('opacity', 0)
              })

          // rework ticks and grid for better visual structure
          function isYear (t) {
            return +t === +(new Date(t.getFullYear(), 0, 1, 0, 0, 0))
          }

          function isMonth (t) {
            return +t === +(new Date(t.getFullYear(), t.getMonth(), 1, 0, 0, 0))
          }

          var xTicks = xScale.ticks()
          var isYearTick = xTicks.map(isYear)
          var isMonthTick = xTicks.map(isMonth)
          // year emphasis
          // ensure year emphasis is only active if years are the biggest clustering unit
          if (emphasizeYearTicks &&
              !(isYearTick.every(function (d) { return d === true })) &&
              isMonthTick.every(function (d) { return d === true })) {
            d3.selectAll('g.tick').each(function (d, i) {
              if (isYearTick[i]) {
                d3.select(this)
                    .attr({
                      'class': 'x_tick_emph'
                    })
              }
            })
            d3.selectAll('.vert_grid').each(function (d, i) {
              if (isYearTick[i]) {
                d3.select(this)
                    .attr({
                      'class': 'vert_grid_emph'
                    })
              }
            })
          }

          // today emphasis
          var todayDate = new Date()
          var emphasizedToday = false
          todayDate.setHours(0, 0, 0, 0)
          d3.selectAll('g.tick').each(function (d, i) {
            if (d.getTime() === todayDate.getTime()) {
              // tick exists for today, emphasizing it
              d3.select(this)
                  .attr({
                    'class': 'x_tick_today'
                  })
            }
          })
          d3.selectAll('.vert_grid').each(function (d, i) {
            if (d.getTime() === todayDate.getTime()) {
              // vertical line exists in the grid for today, emphasizing it
              d3.select(this)
                  .attr({
                    'class': 'vert_grid_today',
                    'stroke-dasharray': '10, 5'
                  })
              emphasizedToday = true
            }
          })

          // not emphasized today yet
          if (emphasizedToday === false) {
            var interpolatedX = xScale.interpolate(d3.interpolate)(todayDate)
            var lastInterpolatedX = xScale.interpolate(d3.interpolate)(endDate)
            if (interpolatedX === lastInterpolatedX && todayDate > endDate) {
              // today is out of range (not displayed in the chart)
              interpolatedX += 18
            }
            // adding the emphasized today vertical line
            svg.select('#g_axis')
                .append('line')
                .attr({
                  'class': 'vert_grid_today',
                  'stroke-dasharray': '10, 5',
                  'x1': interpolatedX,
                  'x2': interpolatedX,
                  'y1': 0,
                  'y2': dataHeight * noOfDatasets + lineSpacing * noOfDatasets - 1 + paddingBottom
                })
          }

          // create title
          if (drawTitle) {
            svg.select('#g_title')
                .append('text')
                .attr('x', paddingLeft)
                .attr('y', paddingTopHeading)
                .text('Data Availability Plot')
                .attr('class', 'heading')
          }

          // create subtitle
          var subtitleText = ''
          if (isDateOnlyFormat) {
            subtitleText = moment(parseDate(startDate)).format('MMMM Y') + ' - ' +
              moment(parseDate(endDate)).format('MMMM Y')
          } else {
            subtitleText = moment(parseDateTime(startDate)).format('l') + ' ' +
                moment(parseDateTime(startDate)).format('LTS') + ' - ' +
                moment(parseDateTime(endDate)).format('l') + ' ' +
                moment(parseDateTime(endDate)).format('LTS')
          }

          svg.select('#g_title')
              .append('text')
              .attr('x', paddingLeft)
              .attr('y', paddingTopHeading + 17)
              .text(subtitleText)
              .attr('class', 'subheading')

          // create legend
          var legend = svg.select('#g_title')
              .append('g')
              .attr('id', 'g_legend')
              .attr('transform', 'translate(0,-12)')

          legend.append('rect')
              .attr('x', width + margin.right - 150 + 50)
              .attr('y', paddingTopHeading)
              .attr('height', 15)
              .attr('width', 15)
              .attr('class', 'rect_has_data')

          legend.append('text')
              .attr('x', width + margin.right - 150 + 70)
              .attr('y', paddingTopHeading + 8.5)
              .text('On time')
              .attr('class', 'legend')

          legend.append('rect')
              .attr('x', width + margin.right - 150 + 50)
              .attr('y', paddingTopHeading + 17)
              .attr('height', 15)
              .attr('width', 15)
              .attr('class', 'rect_has_no_data')

          legend.append('text')
              .attr('x', width + margin.right - 150 + 70)
              .attr('y', paddingTopHeading + 8.5 + 15 + 2)
              .text('Late')
              .attr('class', 'legend')
        })
      }

      chart.width = function (_) {
        if (!arguments.length) return width
        width = _
        return chart
      }

      chart.drawTitle = function (_) {
        if (!arguments.length) return drawTitle
        drawTitle = _
        return chart
      }

      chart.maxDisplayDatasets = function (_) {
        if (!arguments.length) return maxDisplayDatasets
        maxDisplayDatasets = _
        return chart
      }

      chart.curDisplayFirstDataset = function (_) {
        if (!arguments.length) return curDisplayFirstDataset
        curDisplayFirstDataset = _
        return chart
      }

      chart.emphasizeYearTicks = function (_) {
        if (!arguments.length) return emphasizeYearTicks
        emphasizeYearTicks = _
        return chart
      }

      return chart
    },
    refreshChart: function (event) {
      // removing all created taskTooltips to avoid useless scrolling
      var paras = document.getElementsByClassName('taskTooltip')
      while (paras[0]) {
        paras[0].parentNode.removeChild(paras[0])
      }

      var chart = this.visavailChart().width(document.body.clientWidth - 290)

      d3.select('#chart')
        .datum(this.tasks)
        .call(chart)
    }
  },
  mounted: function () {
    // set Moment.js locale and timezone
    moment.locale(process.env.MOMENTJS_LOCALE)
    moment.tz.setDefault(process.env.MOMENTJS_TIMEZONE)

    // refresh the gantt graph
    this.refreshChart()
  }
}
</script>