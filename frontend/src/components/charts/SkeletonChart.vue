<template>

  <div ref="holder">
    <div id="vis"></div>
  </div>
</template>

<script>
import { mapState } from 'vuex'
import AugurStats from '@/AugurStats.ts'
// import {default as vegaEmbed} from 'vega-embed'
// console.log(window)


// vega@5
// vega-lite@3
// vega-embed@4
export default {
  
  props: ['source', 'citeUrl', 'citeText', 'title', 'disableRollingAverage', 'alwaysByDate', 'domain', 'data'],
  data() {

    return {
      legendLabels: [],
      values: [],
      status: {},
      detail: this.$store.state.showDetail,
      compRepos: this.$store.state.comparedRepos,
      metricSource: null,
      timeperiod: 'all',
      forceRecomputeCounter: 0
    }
  },
  computed: {
    repo () {
      return this.$store.state.baseRepo
    },
    gitRepos () {
      return this.$store.state.gitRepo
    },
    period () {
      return this.$store.state.trailingAverage
    },
    earliest () {
      return this.$store.state.startDate
    },
    latest () {
      return this.$store.state.endDate
    },
    compare () {
      return this.$store.state.compare
    },
    comparedRepos () {
      return this.$store.state.comparedRepos
    },
    rawWeekly () {
      return this.$store.state.rawWeekly
    },
    showArea () {
      return this.$store.state.showArea
    },
    showTooltip () {
      return this.$store.state.showTooltip
    },
    showDetail () {
      return this.$store.state.showDetail
    },
    spec() {
      console.log(window)
      // const vegaEmbed = window.vegaEmbed;
      // Get the repos we need
      this.forceRecomputeCounter;
      let repos = []
      if (this.repo) {
        if (window.AugurRepos[this.repo])
          repos.push(window.AugurRepos[this.repo])
        else if (this.domain){
          let temp = window.AugurAPI.Repo({"gitURL": this.gitRepo})
          if (window.AugurRepos[temp])
            temp = window.AugurRepos[temp]
          else
            window.AugurRepos[temp] = temp
          repos.push(temp)
        }
        // repos.push(this.repo)
      } // end if (this.$store.repo)
      this.comparedRepos.forEach(function(repo) {
        repos.push(window.AugurRepos[repo])
      });

      repos.forEach((repo) => {
        this.status[repo] = true
      })

      //COLORS TO PICK FOR EACH REPO
      var colors = ["black", "#FF3647", "#4736FF","#3cb44b","#ffe119","#f58231","#911eb4","#42d4f4","#f032e6"]

      let config = {
        $schema: 'https://vega.github.io/schema/vega-lite/v2.0.json',
        description: 'A simple bar chart with embedded data.',
        data: {
          values: this.data
        },
        mark: 'bar',
        encoding: {
          "x": {"field": "date", "type": "temporal"},
    "y": {"field": "valueRollingrails/rails", "type": "quantitative"},
    "tooltip": {"field": "valueRollingrails/rails", "type": "quantitative"}
        }
      };

      /*
       * Takes a string like "commits,lines_changed:additions+deletions"
       * and makes it into an array of endpoints:
       *
       *   endpoints = ['commits','lines_changed']
       *
       * and a map of the fields wanted from those endpoints:
       *
       *   fields = {
       *     'lines_changed': ['additions', 'deletions']
       *   }
       */
      let endpoints = []
      let fields = {}
      this.source.split(',').forEach((endpointAndFields) => {
        let split = endpointAndFields.split(':')
        endpoints.push(split[0])
        if (split[1]) {
          fields[split[0]] = split[1].split('+')
        }
      })

      let compare = this.compare
      let processData = (data) => {

        // Make it so the user can save the data we are using
          this.__download_data = data
          this.__download_file = this.title.replace(/ /g, '-').replace('/', 'by').toLowerCase()
          // this.$refs.downloadJSON.href = 'data:text/json;charset=utf-8,' + encodeURIComponent(JSON.stringify(this.__download_data))
          // this.$refs.downloadJSON.setAttribute('download', this.__download_file + '.json')


          // We usually want to limit dates and convert the key to being vega-lite friendly
          let defaultProcess = (obj, key, field, count) => {
            let d = null
            if (typeof(field) == "string") {
              field = [field]
            }

            d = AugurStats.convertKey(obj[key], field)
            d = AugurStats.convertDates(d, this.earliest, this.latest, 'date')
            return d
          }

          // Normalize the data into [{ date, value },{ date, value }]
          // BuildLines iterates over the fields requested and runs onCreateData on each
          let normalized = []
          let aggregates = []
          let buildLines = (obj, onCreateData, repo) => {
            if (!obj) {
              return
            }
            if (!onCreateData) {
              onCreateData = (obj, key, field, count) => {
                normalized.push(d)
              }
            }
            let count = 0
            for (var key in obj) {

              if (obj.hasOwnProperty(key)) {
                if (fields[key]) {
                  fields[key].forEach((field) => {
                    onCreateData(obj, key, field, count)
                    count++
                  })
                } else {
                  if (Array.isArray(obj[key]) && obj[key].length > 0) {
                    let field = Object.keys(obj[key][0]).splice(1)
                    onCreateData(obj, key, field, count)
                    count++
                  } else {
                    this.status[repo] = false
                    this.renderError()

                    //return
                  }
                }
              } // end hasOwnProperty
            } // end for in
          } // end normalize function

          // Build the lines we need
          let legend = [] //repo + field strings for vega legend
          let values = []
          let colors = [] 
          let baselineVals = null
          let baseDate = null
          repos.forEach((repo) => {
            // if (this.data){
            //   let temp = {}
            //   temp[this.repo] = {}
            //   temp[this.repo][this.source] = data
            //   data = temp
            // }
            // let relevant = this.data ? data
              buildLines(data[repo], (obj, key, field, count) => {
                // Build basic chart using rolling averages
                let d = defaultProcess(obj, key, field, count)
                
                let rolling = null
                if (repo == this.repo && d[0]) baseDate = d[0].date
                else d = AugurStats.alignDates(d, baseDate, this.period)
                if (this.compare == 'zscore') { // && this.comparedRepos.length > 0
                  rolling = AugurStats.rollingAverage(AugurStats.zscores(d, 'value'), 'value', this.period, repo)
                } //else if (this.rawWeekly || this.disableRollingAverage) rolling = AugurStats.convertKey(d, 'value', 'value' + repo)
                else if (this.compare == 'baseline') { //&& this.comparedRepos.length > 0
                  if(repo.githubURL == this.repo){
                    baselineVals = AugurStats.rollingAverage(d, 'value', this.period, repo)
                  }
                  rolling = AugurStats.rollingAverage(d, 'value', this.period, repo)
                  if(baselineVals){

                    for (var i = 0; i < baselineVals.length; i++){
                    if (rolling[i] && baselineVals[i])
                      rolling[i].valueRolling -= baselineVals[i].valueRolling                   
                    }
                  }
                } else {
                  console.log("rolling")
                  rolling = AugurStats.rollingAverage(d, 'value', this.period, repo)
                }

                normalized.push(AugurStats.standardDeviationLines(rolling, 'valueRolling', repo))
                aggregates.push(AugurStats.convertKey(d, 'value', 'value' + repo))
                legend.push(repo + " " + field)
                colors.push(window.AUGUR_CHART_STYLE.brightColors[count])
              }, repo)

          });

          if (normalized.length == 0) {
            // this.renderError()
          } else {
            values = []
            for(var i = 0; i < legend.length; i++){
              normalized[i].forEach(d => {
                // d.name = legend[i]
                d.color = colors[i]
                values.push(d);
              })
              if (this.rawWeekly) {
                aggregates[i].forEach(d => {
                  // d.name = legend[i]
                  d.color = colors[i]
                  values.push(d)
                })
              }
            }
            repos.forEach((repo) => {
              if(!this.status[repo]) {
                let temp = JSON.parse(JSON.stringify(values))
                temp = temp.map((datum) => {
                  // datum.name = repo + ": data n/a"

                  return datum
                })
                values.push.apply(values, temp)
              }
            })  

            this.legendLabels = legend
            config.data = {"values": values}
            this.values = values

            let allFalse = true
            for(var key in this.status)
              if(this.status[key]) allFalse = false
            if(!allFalse) $(this.$el).find('.error').addClass('hidden')

            // config.config.legend.offset = -(String(this.legendLabels[0]).length * 6.5) - 20
            $(this.$el).find('.hidefirst').removeClass('invis')
            $(this.$el).find('.hidefirst').removeClass('invisDet')
            $(this.$el).find('.spinner').removeClass('loader')
            $(this.$el).find('.spacing').addClass('hidden')
            $(this.$el).find('.hidefirst').removeClass('invisDet')


            //this.mgConfig.legend_target = this.$refs.legend
            this.renderChart()
          }
      }
      if (this.data) {
        processData(this.data)
      } else {
        
        window.AugurAPI.batchMapped(repos, endpoints).then((data) => {
          processData(data)
        }, () => {
          //this.renderError()
        }) // end batch request
      }
      
      // this.reloadImage(config)
  //     vegaEmbed('#viz', config, {actions: false}).then(function(result) {
  //   // Access the Vega view instance (https://vega.github.io/vega/docs/api/view/) as result.view
  //   console.log(result)
  // }).catch(console.error);
  console.log("TEST")

      console.log("CONFIG",config)
      vegaEmbed('#vis', config);

      return config
      
    }

  }, // end computed
  methods: {
    downloadSVG (e) {
      var svgsaver = new window.SvgSaver()
      var svg = window.$(this.$refs.holder).find('svg')[0]
      svgsaver.asSvg(svg, this.__download_file + '.svg')
    },
    downloadPNG (e) {
      var svgsaver = new window.SvgSaver()
      var svg = window.$(this.$refs.holder).find('svg')[0]
      svgsaver.asPng(svg, this.__download_file + '.png')
    },
    renderChart () {
      // this.$refs.chart.className = 'linechart intro'
      // window.$(this.$refs.holder).find('.hideme').removeClass('invis')
      // window.$(this.$refs.holder).find('.showme').removeClass('invis')
      // window.$(this.$refs.holder).find('.hideme').removeClass('invisDet')
      // window.$(this.$refs.holder).find('.showme').removeClass('invisDet')
      // window.$(this.$refs.holder).find('.deleteme').remove()
      // this.$refs.chartholder.innerHTML = ''
      // this.$refs.chartholder.appendChild(this.mgConfig.target)
    },
    renderError () {
        $(this.$el).find('.spinner').removeClass('loader')
        $(this.$el).find('.error').removeClass('hidden')
    },
    thisShouldTriggerRecompute() {
      this.forceRecomputeCounter++;
    },
    reloadImage (config) {
      console.log("reloading")
          vegaEmbed('#' + this.source, config, {actions: false,tooltip: {theme: 'dark'}, mode: 'vega-lite'}) 
    }
  },// end methods
  mounted() {
    this.spec;
  },
  created () {
      
      var query_string = "chart_mapping=" + this.source
      window.AugurAPI.getMetricsStatus(query_string).then((data) => {
        this.metricSource = data[0].data_source
      })
  //     const unwatch = this.$store.watch(
  //   // When the returned result changes...
  //   function (state) {
  //     console.log("WORKED")
  //     return
  //   },
  //   // Run this callback
  //   unwatch()
  // )
  }
}
</script>
