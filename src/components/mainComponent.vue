<template>
  <header>
    chiffre financiers<br/>
    <input  placeholder="siren" v-model="siren"/><br/>
    <input type="button" :disabled="!enableButton" @click="print" value="get me the data">
    <span v-if="loading">Loading…</span>
  </header>
  <main>
    <div v-if="sirenIsSiren && this.result">
      <h2>champs de bilans disponibles pour le SIREN {{ siren }}</h2>
      <div v-for="bilan in result">
        <h2>{{ bilan['date_cloture_exercice'].toLocaleString().slice(0,10) }}</h2>
        <span v-for="(value) in bilan.map">
        <b>{{ value[0] }}</b>: {{ value[1] }}g
        <hr/>
      </span>
      </div>
    </div>
  </main>
</template>

<script lang="ts">
import * as duckdb from '@duckdb/duckdb-wasm';
import duckdb_wasm from '@duckdb/duckdb-wasm/dist/duckdb-mvp.wasm?url';
import mvp_worker from '@duckdb/duckdb-wasm/dist/duckdb-browser-mvp.worker.js?url';
import duckdb_wasm_next from '@duckdb/duckdb-wasm/dist/duckdb-eh.wasm?url';
import eh_worker from '@duckdb/duckdb-wasm/dist/duckdb-browser-eh.worker.js?url';


const MANUAL_BUNDLES = {
  mvp: {
    mainModule: duckdb_wasm,
    mainWorker: mvp_worker,
  },
  eh: {
    mainModule: duckdb_wasm_next,
    // mainWorker: new URL('@duckdb/duckdb-wasm/dist/duckdb-browser-eh.worker.js', import.meta.url).toString(),
    mainWorker: eh_worker
  },
};

const bundle = await duckdb.selectBundle(MANUAL_BUNDLES);

// Instantiate the asynchronus version of DuckDB-wasm
const worker = new Worker(bundle.mainWorker);
const logger = new duckdb.ConsoleLogger();
const db = new duckdb.AsyncDuckDB(logger, worker);
await db.instantiate(bundle.mainModule, bundle.pthreadWorker);

const conn = await db.connect(); // Connect to db

const sirex = /^[0-9]{9}$/
export default {
  name: 'mainComponent',
  data() {
    return {
      result: null,
      loading: false,
      siren: '',
    }
  },
  computed: {
    sirenIsSiren() {
      return ((this.siren.match(sirex)|| []).length > 0)
    },
    enableButton() {
      return !this.loading && this.sirenIsSiren
    },
  },
  methods: {
    print() {
      this.loading = true
      let query = "select * from 'https://static.data.gouv.fr/resources/donnees-financieres-des-entreprises-detaillees-format-parquet/20231224-085013/export-detail-bilans.parquet' where siren='" + this.siren + "'"
      conn.query(query).then((r) => {
        this.result = r
        this.loading = false
      })
    }
  }
}
</script>