<template>
  <v-app>
    <v-container>
      <h1>{{msg}}</h1>
      <div id="graph"></div>
    </v-container>
  </v-app>
</template>

<script>
  import * as d3 from "d3";

  export default {
    name: 'CovidBoard',
    props: {
      msg: String
    },
    mounted() {
      this.generateRace();
    },

    data() {
      return {
        top_n: 12,
        cumulativeByCountry: {},
        entryByDate: {},
        tickDuration: 300,
        height: 600,
        width: 960
      }
    },

    methods: {

      generateRace() {
        
        let svg = d3.select("#graph").append("svg")
          .attr("width", 960)
          .attr("height", 600);

        const margin = {
          top: 80,
          right: 0,
          bottom: 5,
          left: 0
        };

        const barPadding = (this.height - (margin.bottom + margin.top)) / (this.top_n * 5);

        svg.append('text')
          .attr('class', 'title')
          .attr('y', 24)
          .html('Cases until today');

        svg.append("text")
          .attr("class", "subTitle")
          .attr("y", 55)
          .html("COVID-19 cases evolution by country");

        svg.append('text')
          .attr('class', 'caption')
          .attr('x', this.width)
          .attr('y', this.height - 5)
          .style('text-anchor', 'end')
          .html('Source: European Centre for Disease Prevention and Control');

        let startDate = new Date(2019, 11, 31);


        let proxyUrl = 'https://cors-anywhere.herokuapp.com/',
          targetUrl = 'https://opendata.ecdc.europa.eu/covid19/casedistribution/json/';

        d3.json(proxyUrl + targetUrl).then((data) => {

          data = data.records;
          const today = new Date();
          data.forEach(d => {
            d.cases = +d.cases;
            d.day = +d.day;
            d.month = +d.month;
            d.year = +d.year;
            d.countriesAndTerritories = d.countriesAndTerritories + '';
            d.dateRep = new Date(d.year, d.month - 1, d.day);
            d.deaths = +d.deaths;
            d.popData2018 = +d.popData2018;
            d.colour = d3.hsl(Math.random() * 360, 0.75, 0.75);

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
            .range([margin.left, this.width - margin.right - 65]);

          let y = d3.scaleLinear()
            .domain([this.top_n, 0])
            .range([this.height - margin.bottom, margin.top]);

          let xAxis = d3.axisTop()
            .scale(x)
            .ticks(this.width > 500 ? 5 : 2)
            .tickSize(-(this.height - margin.top - margin.bottom))
            .tickFormat(d => d3.format(',')(d));

          svg.append('g')
            .attr('class', 'axis xAxis')
            .attr('transform', `translate(0, ${margin.top})`)
            .call(xAxis)
            .selectAll('.tick line')
            .classed('origin', d => d === 0);

          svg.selectAll('rect.bar')
            .data(dateSlice, d => d.countriesAndTerritories)
            .enter()
            .append('rect')
            .attr('class', 'bar')
            .attr('x', x(0) + 1)
            .attr('width', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) - x(0))
            .attr('y', d => y(d.rank) + 5)
            .attr('height', y(1) - y(0) - barPadding)
            .style('fill', d => d.colour);

          svg.selectAll('text.label')
            .data(dateSlice, d => d.countriesAndTerritories)
            .enter()
            .append('text')
            .attr('class', 'label')
            .attr('x', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) - 8)
            .attr('y', d => y(d.rank) + 5 + ((y(1) - y(0)) / 2) + 1)
            .style('text-anchor', 'end')
            .html(d => d.countriesAndTerritories);

          svg.selectAll('text.valueLabel')
            .data(dateSlice, d => d.countriesAndTerritories)
            .enter()
            .append('text')
            .attr('class', 'valueLabel')
            .attr('x', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) + 5)
            .attr('y', d => y(d.rank) + 5 + ((y(1) - y(0)) / 2) + 1)
            .text(d => d3.format(',.0f')(this.cumulativeByCountry[d.countriesAndTerritories]));

          let yearText = svg.append('text')
            .attr('class', 'yearText')
            .attr('x', this.width - margin.right)
            .attr('y', this.height - 25)
            .style('text-anchor', 'end')
            .html(this.displayableDate(startDate))
            .call(halo, 10);

          let ticker = d3.interval(() => {
            let dateSlice = this.getSanitizedData(this.entryByDate[startDate], this.cumulativeByCountry);

            x.domain([0, d3.max(dateSlice, d => this.cumulativeByCountry[d.countriesAndTerritories])]);

            svg.select('.xAxis')
              .transition()
              .duration(this.tickDuration)
              .ease(d3.easeLinear)
              .call(xAxis);

            let bars = svg.selectAll('.bar').data(dateSlice, d => d.countriesAndTerritories);

            bars
              .enter()
              .append('rect')
              .attr('class', d => `bar ${d.countriesAndTerritories.replace(/\s/g, '_')}`)
              .attr('x', x(0) + 1)
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

            let labels = svg.selectAll('.label')
              .data(dateSlice, d => d.countriesAndTerritories);

            labels
              .enter()
              .append('text')
              .attr('class', 'label')
              .attr('x', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) - 8)
              .attr('y', () => y(this.top_n + 1) + 5 + ((y(1) - y(0)) / 2))
              .style('text-anchor', 'end')
              .html(d => d.countriesAndTerritories)
              .transition()
              .duration(this.tickDuration)
              .ease(d3.easeLinear)
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


            let valueLabels = svg.selectAll('.valueLabel').data(dateSlice, d => d.countriesAndTerritories);

            valueLabels
              .enter()
              .append('text')
              .attr('class', 'valueLabel')
              .attr('x', d => x(this.cumulativeByCountry[d.countriesAndTerritories]) + 5)
              .attr('y', () => y(this.top_n + 1) + 5)
              .text(d => d3.format(',.0f')(this.cumulativeByCountry[d.countriesAndTerritories]))
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

            if (today.getDate() === startDate.getDate()
              && today.getMonth() === startDate.getMonth()
              && today.getFullYear() === startDate.getFullYear())
              ticker.stop();
            startDate.setDate(startDate.getDate() + 1);
          }, this.tickDuration);

        });

        const halo = function (text, strokewidth) {
          text.select(function () {
            return this.parentNode.insertBefore(this.cloneNode(true), this);
          })
            .style('fill', '#ffffff')
            .style('stroke', '#ffffff')
            .style('stroke-width', strokewidth)
            .style('stroke-linejoin', 'round')
            .style('opacity', 1);

        }
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
    opacity: 0.25;
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
