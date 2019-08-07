<template>
  <div class='demo_container'>
    <section class="button_sec">
      <button v-on:click="pop_pie_clicked">Draw Population Pie</button>
      <button v-on:click="income_pie_clicked">Draw Income Pie</button>
      <section class="input_sec">
        <span>Slice Index: </span>
        <input type="number" min=0 max=6 v-on:change="slice_index_changed($event)"/>
      </section>
      <span class="output_box">{{current_value}}</span>
    </section>
    <pie-chart-comp-d3
        title_1="Distribution Across Ages"
        :title_2="title_2"
        :inner_radius=inner_radius
        value_format=".3s"
        :show_legend="show_legend"
        :chart_data="chart_data"
        :keys="keys"
        :slice_index="slice_index"
        :css_variables="css_variables"
        v-on:piechartcompd3_slice_clicked="value => set_current(value)">
      <svg class="svg_chart_1"></svg>
    </pie-chart-comp-d3>
  </div>
</template>

<script>
  import PieChartCompD3 from 'piechartcompd3';
  import {read_csv} from 'd3fetchmodule';

  export default {
    name: 'App',
    data: function() {
      return {
        chart_data: null,
        title_2: null,
        show_legend: true,
        keys: null,
        current_value: null,
        slice_index: null,
        inner_radius: 80,

        css_variables: {
          pie_chart_compD3_color: 'white',
          pie_chart_compD3_background_color: 'black',
          pie_chart_compD3_slice_text_color: 'gold',
          pie_chart_compD3_tooltip_color: 'white',

          table_comp_thead_color: 'white',
          table_comp_thead_background: 'black',
          table_comp_thead_border_bottom: '2px solid white',
          table_comp_row_color: 'white',
          table_comp_row_selected_color: 'gold',
          table_comp_row_border_bottom: '1px solid white',
          table_comp_row_odd_background: 'black',
          table_comp_row_even_background: 'black'
        }
      };
    },
    components: {
      PieChartCompD3
    },
    methods: {
      pop_pie_clicked: function(){
        this.keys = {text_key: 'age',value_key: 'population',color_key: 'color'};
        this.title_2 = '(for population)';
        const convert = [
          {field: 'population', type: 'linear'},
        ];
        const get_data = async () => {
          try{
            this.chart_data = await read_csv('data/ages_population.csv', convert);
            //debug
            console.log(JSON.stringify(this.chart_data));
          }catch(e){
            console.log(e);
          }
        };
        get_data();
      },
      income_pie_clicked: function(){
        this.keys = {text_key: 'age',value_key: 'income',color_key: 'color'};
        this.title_2 = '(for income)';
        const convert = [
          {field: 'income', type: 'linear'},
        ];
        const get_data = async () => {
          try{
            this.chart_data = await read_csv('data/ages_average_income.csv', convert);
            //debug
            console.log(JSON.stringify(this.chart_data));
          }catch(e){
            console.log(e);
          }
        };
        get_data();
      },
      slice_index_changed: function(e){
        const idx = parseInt(e.currentTarget.value);
        if(idx < 7){
          this.slice_index = idx;
        }
      },
      set_current: function(value){
        this.current_value = `${value.slice_text} = ${value.slice_value}`;
      }
    }
  }
</script>

<style>
  .demo_container {
    background-color: black;
    padding: 20px;
  }
  .button_sec {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    color: white;
    width: 600px;

  }
</style>