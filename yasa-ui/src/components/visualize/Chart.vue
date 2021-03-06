<template>
  <el-container id="chart-root">
    <el-header height="28px">
      <el-input v-model="localQueryString" :placeholder="$t('discover.queryInputHint')" @keyup.enter.native="onQuery">
        <el-select slot="prepend" v-model="collection" :value="collection" :loading="$store.state.loadingCollections">
          <el-option v-for="c in collections" :key="c" :value="c"></el-option>
        </el-select>
        <el-button slot="append" icon="el-icon-search" @click="onQuery" :loading="loadingChartData">{{
          $t('discover.numHit', [numHit])
        }}</el-button>
      </el-input>
    </el-header>
    <el-container>
      <el-aside>
        <chart-form :fields="fields" :loading-fields="loadingFields" @submit="onSubmit"></chart-form>
      </el-aside>
      <el-main>
        <chart class="chart" ref="chart" :options="chartOptions" auto-resize />
      </el-main>
    </el-container>
  </el-container>
</template>

<script lang="ts">
import { Component, Vue, Watch } from 'vue-property-decorator';
import { namespace, State } from 'vuex-class';
import { Field } from '@/model';
import { Bucket, SelectResult } from '@service/solr/collections';
import ChartForm from './ChartForm.vue';
import { ChartFormData } from '@store/modules/visualize/ChartFormData';

const Store = namespace('visualize');

@Component({
  components: {
    ChartForm,
  },
})
export default class YasaChart extends Vue {
  private localQueryString = '';

  @State private collections!: string[];

  @Store.State private loadingFields!: boolean;
  @Store.State private fields!: Field[];
  @Store.State private chartDataSource!: { [title: string]: Bucket[] };
  @Store.State private loadingChartData!: boolean;
  @Store.State private result!: SelectResult;
  @Store.State private formData!: ChartFormData;

  @Store.Mutation private setCollection!: (collection: string) => void;
  @Store.Mutation private setFormData!: (formData: ChartFormData) => void;
  @Store.Mutation private setQueryString!: (queryString: string) => void;
  @Store.Mutation private setResult!: (result: SelectResult) => void;

  @Store.Action private loadFields!: () => void;
  @Store.Action private loadChartData!: () => void;
  @Store.Action private reset!: () => void;

  created() {
    this.selectCollection();
  }

  mounted() {
    this.reset();
  }

  private get collection() {
    return this.$store.state.visualize.collection;
  }

  private set collection(val) {
    this.setCollection(val);
  }

  private get chartType() {
    const type = this.formData.type;
    switch (type) {
      case 'line':
      case 'area':
        return 'line';
      case 'bar':
        return 'bar';
      default:
        return 'line';
    }
  }

  private areaStyle() {
    const type = this.formData.type;
    return type === 'area' ? {} : undefined;
  }

  private get chartOptions() {
    return {
      tooltip: {
        trigger: 'axis',
        axisPointer: {},
      },
      legend: {},
      xAxis: {
        type: 'category',
        axisTick: {
          alignWithLabel: true,
        },
        axisLabel: {
          rotate: 45,
        },
      },
      yAxis: {},
      series: this.formData.charts.map((chart) => ({
        name: chart.title,
        type: chart.type,
        data: (this.chartDataSource[chart.title] || []).map((bucket) => ({
          name: bucket.val,
          value: bucket.count,
        })),
        dimensions: ['val', 'yAxis'],
        areaStyle: this.areaStyle,
        smooth: true,
      })),
      grid: {
        left: 10,
        right: 20,
        bottom: 20,
        top: 40,
        containLabel: true,
      },
    };
  }

  private get numHit() {
    return ((this.result || {}).response || {}).numFound || 0;
  }

  private onSubmit(formData: ChartFormData) {
    this.setFormData(formData);
    this.loadChartData();
  }

  private onQuery() {
    this.setQueryString(this.localQueryString);
    this.loadChartData();
  }

  private selectCollection() {
    const lastCollection = localStorage.getItem('collection');
    if (lastCollection && this.collections.includes(lastCollection)) {
      this.setCollection(lastCollection);
    } else if (this.collections.length > 0) {
      this.setCollection(this.collections[0]);
    }
  }

  @Watch('collections')
  private onCollectionsChanged() {
    this.selectCollection();
  }

  @Watch('loadingChartData')
  private onLoadingChartDataChanged() {
    type ECharts = Vue & {
      showLoading: () => void;
      hideLoading: () => void;
    };
    if (this.loadingChartData) {
      (this.$refs.chart as ECharts).showLoading();
    } else {
      (this.$refs.chart as ECharts).hideLoading();
    }
  }

  @Watch('collection', { immediate: true })
  private onCollectionChanged() {
    this.setFormData(new ChartFormData());
    this.setResult({} as SelectResult);
    this.loadFields();
  }
}
</script>

<style scoped>
#chart-root {
  height: 100%;
  overflow: auto;
  padding: 0;
}
.el-main,
.el-container,
.el-header {
  height: 100%;
  padding: 0;
}
.el-header {
  padding: 0 8px;
}
.el-aside {
  padding: 8px;
  border-right: 1px solid lightgray;
}
.el-select {
  width: 290px;
}
.chart {
  width: 100%;
  height: 100%;
}
</style>
