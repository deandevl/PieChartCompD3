<template>
  <div>
    <div class="pieChartCompD3" :style="chart_width_style">
      <div class="pieChartCompD3_chart_box">
        <section class="pieChartCompD3_chart_box__svg">
          <slot></slot>
        </section>
        <section class="pieChartCompD3_chart_box__legend" v-show="show_legend">
          <table-comp
              :headings="legend_headings"
              :rows="legend_rows"
              :column_widths="legend_column_widths"
              :cell_styles="cell_styles"
              v-on:table_comp_row="table_row_clicked">
          </table-comp>
        </section>
      </div>
    </div>
  </div>
</template>

<script>
  import {arc,pie} from 'd3-shape';
  import {select} from 'd3-selection';
  import {format} from 'd3-format';
  import TableComp from 'tablecomp';
  import {color_scale} from 'd3ScaleModule';
  import {make_draggable} from 'd3utilmodule';

  export default {
    name: 'PieChartCompD3',
    data(){
      return {
        data_with_percent: null,
        path_generator: null,
        pie_function: null,
        pixels: {x: null,y: null},
        chart_g: null,
        legend_headings: null,
        legend_column_widths: [40,160,120,120],
        legend_rows: null,
        cell_styles: null,
        color_scale: null,
        tooltip_text: null,
        current_slice: null
      }
    },
    props: {
      chart_data: {
        type: Array,
        default: null
      },
      keys: {
        type: Object,
        default: null
      },
      slice_index: {
        type: Number,
        default: null
      },
      value_format: {
        type: String,
        default: null
      },
      outer_radius: {
        type: Number,
        default: 200
      },
      inner_radius: {
        type: Number,
        default: 0
      },
      title_1: {
        type: String,
        default: null
      },
      title_2: {
        type: String,
        default: null
      },
      margin_top: {
        type: Number,
        default: 90
      },
      margin_left: {
        type: Number,
        default: 60
      },
      margin_bottom: {
        type: Number,
        default: 40
      },
      show_legend: {
        type: Boolean,
        default: true
      },
      css_variables: {
        type: Object,
        default: null
      }
    },
    watch: {
      chart_data: function(new_data){
        if(new_data !== null){
          //define per cent value
          //get total
          let total = 0;
          new_data.forEach(d => {
            total += d[this.keys.value_key];
          });

          this.data_with_percent = new_data.map(d => {
            return {
              slice_text: d[this.keys.text_key],
              slice_value: d[this.keys.value_key],
              slice_percent: d[this.keys.value_key] / total,
              slice_color: d[this.keys.color_key]
            };
          });

          //define ordinal color scale
          this.color_scale = color_scale(new_data.map(d => {
            return d[this.keys.color_key];
          }));

          //define headings for legend table
          this.legend_headings = ['slice', this.keys.text_key, this.keys.value_key, 'percent'];

          //define pie_data
          const pie_data = this.pie_function(this.data_with_percent);

          //remove all current elements
          this.remove_elements();

          //draw pie
          this.draw_pie(pie_data);

          //draw main titles
          this.draw_main_titles();

          //update legend table
          this.update_legend_table();

          //create text toottip
          this.create_tooltip();
        }
      },
      slice_index: function(i){
        const cell_styles = this.cell_styles.slice(0);
        const slice = this.data_with_percent[i];
        if(this.current_slice !== null){
          this.current_slice.g.style('opacity',.7);
          cell_styles[this.current_slice.idx][0] = `background-color: ${this.current_slice.color};opacity: 0.7;`;
        }
        //highlight slice selection with change in opacity
        const slice_g = this.chart_g.select('.pieChartCompD3_pie_g').select('.pieChartCompD3_slice_' + i);
        this.current_slice = {g: slice_g, idx: i, color: slice.slice_color};
        slice_g.style('opacity',1.0);
        //highlight legend table as well
        cell_styles[i][0] = `background-color: ${slice.slice_color};opacity: 1.0;`;
        this.cell_styles = cell_styles;
        this.$emit('piechartcompd3_slice_clicked',
          {
            index: i,
            slice_text: slice.slice_text,
            slice_value: slice.slice_value,
            slice_percent: slice.slice_percent,
            slice_color: slice.slice_color
          });
      }
    },
    computed: {
      chart_width_style: function(){
        return `width:${this.margin_left+this.outer_radius*2}px`;
      }
    },
    methods: {
      draw_pie: function(pie_data){
        const pie_g = this.chart_g.append('g')
          .attr('class','pieChartCompD3_pie_g')
          .attr('transform',`translate(${this.outer_radius+this.margin_left},${this.outer_radius+this.margin_top})`);

        const arc_groups = pie_g.selectAll('g')
          .data(pie_data)
          .enter().append('g')
          .attr('class',(d,i) => {
            return 'pieChartCompD3_slice pieChartCompD3_slice_' + i;
          })
          .on('mouseover',d => {
            const loc = this.path_generator.centroid(d);
            this.tooltip_text
              .attr('x',this.outer_radius+this.margin_left+loc[0])
              .attr('y',this.outer_radius+this.margin_top+loc[1]+15)
              .text(format(this.value_format)(d.value));
          })
          .on('mouseout',() => {
            this.tooltip_text.text('');
          })
          .on('click',(d,i) => {
            this.update_highlight(i);
          });

        //define arc line paths
        arc_groups.append('path')
          .attr('d',this.path_generator)
          .style('fill',(d,i) => {
            return this.color_scale(i);
          });

        //generate text labels for each slice
        arc_groups.append('text')
          .attr('class','pieChartCompD3_slice_text')
          .attr('transform',d => {
            return `translate(${this.path_generator.centroid(d)})`;
          })
          .attr('text-anchor','middle')
          .text(d => {
            return d.data.slice_text;
          });
      },
      draw_main_titles: function(){
        const title_1_g = this.chart_g.append('g')
          .attr('class','pieChartCompD3_title_g_1');
        title_1_g.append('text')
          .attr('class','pieChartCompD3_title')
          .attr('transform',`translate(${this.margin_left+this.outer_radius},30)`)
          .attr('text-anchor','middle')
          .style('font-size','1.3rem')
          .text(this.title_1);
        make_draggable(title_1_g);

        const title_2_g = this.chart_g.append('g')
          .attr('class','pieChartCompD3_title_g_2');
        title_2_g.append('text')
          .attr('class','pieChartCompD3_title')
          .attr('transform',`translate(${this.margin_left+this.outer_radius},50)`)
          .attr('text-anchor','middle')
          .style('font-size','.8rem')
          .text(this.title_2);
        make_draggable(title_2_g);
      },
      create_tooltip: function(){
        this.tooltip_text = this.chart_g.append('text')
          .attr('class', 'pieChartCompD3_tooltip')
          .attr('x',0)
          .attr('y',0)
          .text('');
      },
      update_legend_table: function(){
        this.legend_rows = [];
        this.cell_styles = [];

        this.data_with_percent.forEach(d => {
          const row = [' '];
          row.push(d.slice_text);
          row.push(d.slice_value);
          row.push((d.slice_percent * 100).toFixed(2));
          this.legend_rows.push(row);
          this.cell_styles.push([`background-color: ${d.slice_color};opacity: 0.7;`,'text-align: center;','text-align: center;','text-align: center;']);
        });
      },
      table_row_clicked: function(obj){
        this.update_highlight(obj.row_index);
      },
      update_highlight: function(idx){
        const slice_text = this.data_with_percent[idx].slice_text;
        const slice_value = this.data_with_percent[idx].slice_value;
        const slice_percent = this.data_with_percent[idx].slice_percent;
        const slice_color = this.data_with_percent[idx].slice_color;
        const cell_styles = this.cell_styles.slice(0);
        if(this.current_slice !== null){
          this.current_slice.g.style('opacity',.7);
          //update legend table's current row index style
          cell_styles[this.current_slice.idx][0] = `background-color: ${this.current_slice.color};opacity: 0.7;`;
        }
        //highlight slice selection with change in opacity
        const slice_g = this.chart_g.select('.pieChartCompD3_pie_g').select('.pieChartCompD3_slice_' + idx);
        slice_g.style('opacity',1.0);
        //update this.current_slice
        this.current_slice = {g: slice_g, idx: idx};
        //update legend table's new current row index style as well
        cell_styles[idx][0] = `background-color: ${slice_color};opacity: 1.0;`;
        //make table react with new cell_styles array
        this.cell_styles = cell_styles;
        //broadcast event about new current slice
        this.$emit('piechartcompd3_slice_clicked',
          {
            index: idx,
            slice_text: slice_text,
            slice_value: slice_value,
            slice_percent: slice_percent,
            slice_color: slice_color
          });
      },
      remove_elements: function(){
        //remove main pie group
        this.chart_g.select('.pieChartCompD3_pie_g').remove();
        //remove main titles
        this.chart_g.select('.pieChartCompD3_title_g_1').remove();
        this.chart_g.select('.pieChartCompD3_title_g_2').remove();
        //remove tooltip
        this.chart_g.select('.pieChartCompD3_tooltip').remove();
      }
    },
    components: {
      TableComp
    },
    mounted(){
      //locate svg
      const svg_class = this.$slots.default[0].elm.classList[0];
      const svg = select(`.${svg_class}`);
      svg.attr('width',2*this.outer_radius+this.margin_left);
      svg.attr('height',2*this.outer_radius+this.margin_top+this.margin_bottom);
      this.chart_g = svg.append('g')
        .attr('class','chart_g');

      if(this.css_variables !== null){
        for(let key of Object.keys(this.css_variables)){
          this.$el.style.setProperty(`--${key}`, this.css_variables[key]);
        }
      }

      //define path_generator
      this.path_generator =
        arc()
          .innerRadius(this.inner_radius)
          .outerRadius(this.outer_radius);

      //define pie_function
      this.pie_function =
        pie()
          .value(d => {
            return d.slice_value;
          })
          .sort(null);
    }
  }
</script>

<style lang="less">
  :root {
    --pie_chart_compD3_font_family: Verdana,serif;
    --pie_chart_compD3_color: black;
    --pie_chart_compD3_background_color: white;

    --pie_chart_compD3_slice_text_color: black;
    --pie_chart_compD3_tooltip_color: black;
  }

  .pieChartCompD3 {
    display: flex;
    flex-direction: column;
    font-family: var(--pie_chart_compD3_font_family);
    color: var(--pie_chart_compD3_color);
    background-color: var(--pie_chart_compD3_background_color);

    &_title {
      font-weight: bold;
      fill: var(--pie_chart_compD3_color);
      cursor: move;
    }

    &_slice {
      opacity: .7;
    }

    &_slice_text {
      font-weight: bold;
      font-family: var(--pie_chart_compD3_font_family);
      fill: var(--pie_chart_compD3_slice_text_color);
    }

    &_tooltip {
      text-anchor: middle;
      font-weight: bold;
      font-size: 12px;
      font-family: var(--pie_chart_compD3_font_family);
      fill: var(--pie_chart_compD3_tooltip_color)
    }

    &_chart_box {
      display: flex;
      flex-direction: row;

      &__legend {
        margin-left: .8rem;
        align-self: center;
      }
    }
  }
</style>