<template>
  <v-app>
    <v-container>
      <h1>{{welcome}}</h1>
      <div id="graph"></div>
      <div class="text-center">
        <v-btn v-on:click="launchRace" rounded color="primary" dark>Replay</v-btn>
      </div>
    </v-container>
  </v-app>
</template>

<script>
  import * as d3 from "d3";

  export default {
    name: 'CovidBoard',
    props: {
      welcome: String
    },
    mounted() {
      this.generateRace();
    },

    data() {
      return {
        top_n: 12,
        cumulativeByCountry: {},
        entryByDate: {},
        tickDuration: 600,
        height: 600,
        width: 960,
        margin: {
          top: 80,
          right: 0,
          bottom: 5,
          left: 0
        },
        ticker: null,
        svg: null
      }
    },

    methods: {

      generateRace() {

        this.svg = d3.select("#graph").append("svg")
          .attr("width", 960)
          .attr("height", 600);

        this.svg.append('text')
          .attr('class', 'title')
          .attr('y', 24)
          .style('fill', 'white')
          .html('Cases until today');

        this.svg.append("text")
          .attr("class", "subTitle")
          .attr("y", 55)
          .style('fill', 'white')
          .html("COVID-19 cases evolution by country");

        this.svg.append('text')
          .attr('class', 'caption')
          .attr('x', this.width)
          .attr('y', this.height - 5)
          .style('fill', 'white')
          .style('text-anchor', 'end')
          .html('Source: European Centre for Disease Prevention and Control');

        let proxyUrl = 'https://cors-anywhere.herokuapp.com/',
          targetUrl = 'https://opendata.ecdc.europa.eu/covid19/casedistribution/json/';

        d3.json(proxyUrl + targetUrl).then((data) => {
          this.launchRace(data.records);
        });
      },

      launchRace(data) {
        let startDate = new Date(2019, 11, 31);
        const barPadding = (this.height - (this.margin.bottom + this.margin.top)) / (this.top_n * 5);
        const today = new Date();
        data.forEach(d => {
          d.cases = +d.cases;
          d.day = +d.day;
          d.month = +d.month;
          d.year = +d.year;
          d.countriesAndTerritories = d.countriesAndTerritories + '';
          if(d.countriesAndTerritories === "Cases_on_an_international_conveyance_Japan"){
            d.countriesAndTerritories = 'Diamond Princess - Cruise Ship';
          }
          d.dateRep = new Date(d.year, d.month - 1, d.day);
          d.deaths = +d.deaths;
          d.popData2018 = +d.popData2018;
          d.colour = d3.hsl(Math.random() * 360, 0.8, 0.5);

          if (d.dateRep in this.entryByDate) {
            let prev = this.entryByDate[d.dateRep];
            prev.push(d);
            this.entryByDate[d.dateRep] = prev;
          } else {
            this.entryByDate[d.dateRep] = [d];
          }
        });

        let dateSlice = this.getSanitizedData(this.entryByDate[startDate], this.cumulativeByCountry);

        let x = d3.scaleLinear()
          .domain([0, d3.max(dateSlice, d => this.cumulativeByCountry[d.countriesAndTerritories])])
          .range([this.margin.left, this.width - this.margin.right - 65]);

        let y = d3.scaleLinear()
          .domain([this.top_n, 0])
          .range([this.height - this.margin.bottom, this.margin.top]);

        let xAxis = d3.axisTop()
          .scale(x)
          .ticks(this.width > 500 ? 5 : 2)
          .tickSize(-(this.height - this.margin.top - this.margin.bottom))
          .tickFormat(d => d3.format(',')(d));

        this.svg.append('g')
          .attr('class', 'axis xAxis')
          .attr('x', 20)
          .attr('transform', `translate(0, ${this.margin.top})`)
          .call(xAxis)
          .selectAll('.tick line')
          .classed('origin', d => d === 0);

        this.svg.selectAll('rect.bar')
          .data(dateSlice, d => d.countriesAndTerritories)
          .enter()
          .append('rect')
          .attr('class', 'bar')
          .attr('x', 0)
          .attr('width', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) - x(0))
          .attr('y', d => y(d.rank) + 5)
          .attr('height', y(1) - y(0) - barPadding)
          .style('fill', d => d.colour);

        this.svg.selectAll('text.label')
          .data(dateSlice, d => d.countriesAndTerritories)
          .enter()
          .append('text')
          .attr('class', 'label')
          .attr('x', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) - 8)
          .attr('y', d => y(d.rank) + 5 + ((y(1) - y(0)) / 2) + 1)
          .style('text-anchor', 'end')
          .style('fill', 'white')
          .html(d => d.countriesAndTerritories.replace(/_/g, ' '));

        this.svg.selectAll('text.valueLabel')
          .data(dateSlice, d => d.countriesAndTerritories)
          .enter()
          .append('text')
          .attr('class', 'valueLabel')
          .attr('x', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) + 5)
          .attr('y', d => y(d.rank) + 5 + ((y(1) - y(0)) / 2) + 1)
          .style('fill', 'white')
          .text(d => d3.format(',.0f')(this.cumulativeByCountry[d.countriesAndTerritories]));

        let yearText = this.svg.append('text')
          .attr('class', 'yearText')
          .attr('x', this.width - this.margin.right)
          .attr('y', this.height - 25)
          .style('text-anchor', 'end')
          .style('fill', 'white')
          .html(this.displayableDate(startDate));

        this.ticker = d3.interval(() => {
          let dateSlice = this.getSanitizedData(this.entryByDate[startDate], this.cumulativeByCountry);

          x.domain([0, d3.max(dateSlice, d => this.cumulativeByCountry[d.countriesAndTerritories])]);

          this.svg.select('.xAxis')
            .transition()
            .duration(this.tickDuration)
            .ease(d3.easeLinear)
            .call(xAxis);

          let bars = this.svg.selectAll('.bar').data(dateSlice, d => d.countriesAndTerritories);

          bars
            .enter()
            .append('rect')
            .attr('class', d => `bar ${d.countriesAndTerritories.replace(/\s/g, '_')}`)
            .attr('x', 0)
            .attr('width', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) - x(0))
            .attr('y', () => y(this.top_n + 1) + 5)
            .attr('height', y(1) - y(0) - barPadding)
            .style('fill', d => d.colour)
            .transition()
            .duration(this.tickDuration)
            .ease(d3.easeLinear)
            .attr('y', d => y(d.rank) + 5);

          bars
            .transition()
            .duration(this.tickDuration)
            .ease(d3.easeLinear)
            .attr('width', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) - x(0) - 1)
            .attr('y', d => y(d.rank) + 5);

          bars
            .exit()
            .transition()
            .duration(this.tickDuration)
            .ease(d3.easeLinear)
            .attr('width', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) - x(0) - 1)
            .attr('y', () => y(this.top_n + 1) + 5)
            .remove();

          let labels = this.svg.selectAll('.label')
            .data(dateSlice, d => d.countriesAndTerritories);

          labels
            .enter()
            .append('text')
            .attr('class', 'label')
            .attr('x', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) - 8)
            .attr('y', () => y(this.top_n + 1) + 5 + ((y(1) - y(0)) / 2))
            .style('text-anchor', 'end')
            .html(d => d.countriesAndTerritories.replace(/_/g, ' '))
            .transition()
            .duration(this.tickDuration)
            .ease(d3.easeLinear)
            .style('fill', 'white')
            .attr('y', d => y(d.rank) + 5 + ((y(1) - y(0)) / 2) + 1);


          labels
            .transition()
            .duration(this.tickDuration)
            .ease(d3.easeLinear)
            .attr('x', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) - 8)
            .attr('y', d => y(d.rank) + 5 + ((y(1) - y(0)) / 2) + 1);

          labels
            .exit()
            .transition()
            .duration(this.tickDuration)
            .ease(d3.easeLinear)
            .attr('x', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) - 8)
            .attr('y', () => y(this.top_n + 1) + 5)
            .remove();


          let valueLabels = this.svg.selectAll('.valueLabel').data(dateSlice, d => d.countriesAndTerritories);

          valueLabels
            .enter()
            .append('text')
            .attr('class', 'valueLabel')
            .attr('x', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) + 5)
            .attr('y', () => y(this.top_n + 1) + 5)
            .text(d => d3.format(',.0f')(this.cumulativeByCountry[d.countriesAndTerritories]))
            .style('fill', 'white')
            .transition()
            .duration(this.tickDuration)
            .ease(d3.easeLinear)
            .attr('y', d => y(d.rank) + 5 + ((y(1) - y(0)) / 2) + 1);

          valueLabels
            .transition()
            .duration(this.tickDuration)
            .ease(d3.easeLinear)
            .attr('x', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) + 5)
            .attr('y', d => y(d.rank) + 5 + ((y(1) - y(0)) / 2) + 1)
            .tween("text", (d) => {
              let i = d3.interpolateRound(this.cumulativeByCountry[d.countriesAndTerritories], this.cumulativeByCountry[d.countriesAndTerritories]);
              return function (t) {
                this.textContent = d3.format(',')(i(t));
              };
            });

          valueLabels
            .exit()
            .transition()
            .duration(this.tickDuration)
            .ease(d3.easeLinear)
            .attr('x', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) + 5)
            .attr('y', () => y(this.top_n + 1) + 5)
            .remove();

          yearText.html(this.displayableDate(startDate));

          if (8 === startDate.getDate()
            && 1 === startDate.getMonth()
            && today.getFullYear() === startDate.getFullYear())
            this.ticker.stop();
          startDate.setDate(startDate.getDate() + 1);
        }, this.tickDuration);
      },

      displayableDate(startDate) {
        return startDate.getDate() + '/' + parseInt(startDate.getMonth() + 1) + '/' + startDate.getFullYear()
      },

      getSanitizedData(dateEntries) {
        dateEntries.forEach((d) => {
          if (d.countriesAndTerritories in this.cumulativeByCountry) {
            this.cumulativeByCountry[d.countriesAndTerritories] = d.cases + this.cumulativeByCountry[d.countriesAndTerritories]
          } else {
            this.cumulativeByCountry[d.countriesAndTerritories] = d.cases
          }
        });

        // exclude China metrics
        this.cumulativeByCountry['China'] = 0;

        let dateSlice = dateEntries.sort((a, b) => {
          return this.cumulativeByCountry[b.countriesAndTerritories] - this.cumulativeByCountry[a.countriesAndTerritories]
        }).slice(0, this.top_n);

        dateSlice.forEach((d, i) => d.rank = i);

        return dateSlice
      }
    }
  }
</script>
<style>


  h1 {
    color: white;
  }
  text {
    font-size: 16px;
    font-family: Open Sans, sans-serif;
  }

  text.title {
    font-size: 24px;
    font-weight: 500;
  }

  text.subTitle {
    font-weight: 500;
    fill: #777777;
  }

  text.caption {
    font-weight: 400;
    font-size: 14px;
    fill: #777777;
  }

  text.label {
    font-weight: 600;
  }

  text.valueLabel {
    font-weight: 300;
  }

  text.yearText {
    font-size: 64px;
    font-weight: 700;
    opacity: 0.50;
  }

  .tick text {
    fill: #777777;
  }

  .xAxis .tick:nth-child(2) text {
    text-anchor: start;
  }

  .tick line {
    shape-rendering: CrispEdges;
    stroke: #dddddd;
  }

  .tick line.origin {
    stroke: #aaaaaa;
  }

  path.domain {
    display: none;
  }
</style>
