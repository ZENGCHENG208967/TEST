<template>
  <div>
    <a-card class="ant-card-shadow">
      <template #title>
        <h3>
         I sell the Product
        </h3>
      </template>
      <a-table :columns="columns" :loading="state.loading" :data-source="state.init">
        <template #time="{text, record}">
          {{new Date(text * 1000).toLocaleString()}}
        </template>
        <template #tag="{text, record}">
          <a-tag color="success" v-if="record.success === true">
            <template #icon>
              <check-circle-outlined />
            </template>
            Purchase success
          </a-tag>
          <a-tag color="processing" v-else-if="new Date(record.endTime * 1000) > new Date()" >
            <template #icon>
              <sync-outlined :spin="true" />
            </template>
            waiting Purchase
          </a-tag>
          <a-tag color="error" v-else>
            <template #icon>
              <close-circle-outlined />
            </template>
            Purchase failure
          </a-tag>
        </template>
        <template #action="{text, record}">
          <a @click="clickFunding(record.index)">View details</a>
        </template>
      </a-table>
    </a-card>

    <a-card class="ant-card-shadow" style="margin-top: 1em;">
      <template #title>
        <h3>
          I Purchased a Product
        </h3>
      </template>
      <a-table :columns="columns" :loading="state.loading" :data-source="state.contr">
        <template #time="{text, record}">
          {{new Date(text * 1000).toLocaleString()}}
        </template>
        <template #tag="{text, record}">
          <a-tag color="success" v-if="record.success === true">
            <template #icon>
              <check-circle-outlined />
            </template>
            Purchase success
          </a-tag>
          <a-tag color="processing" v-else-if="new Date(record.endTime * 1000) > new Date()" >
            <template #icon>
              <sync-outlined :spin="true" />
            </template>
            waiting Purchase
          </a-tag>
          <a-tag color="error" v-else>
            <template #icon>
              <close-circle-outlined />
            </template>
            Purchase failure
          </a-tag>
        </template>
        <template #action="{text, record}">
          <a @click="clickFunding(record.index)">View details</a>
        </template>
      </a-table>
    </a-card>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, reactive } from 'vue';
import Modal from '../components/base/modal.vue'
import CreateForm from '../components/base/createForm.vue'
import { Model, Fields, Form } from '../type/form'
import { contract, getAccount, getAllFundings, Funding, newFunding, getMyFundings, addListener } from '../api/contract'
import { message } from 'ant-design-vue'
import { CheckCircleOutlined, SyncOutlined, CloseCircleOutlined } from '@ant-design/icons-vue'
import { useRouter } from 'vue-router'

const columns = [
  {
    dataIndex: 'title',
    key: 'title',
    title: 'Product title'
  },
  {
    title: 'Price(eth)',
    dataIndex: 'goal',
    key: 'goal'
  },
  {
    title: 'Current amount(eth)',
    dataIndex: 'amount',
    key: 'amount'
  },
  {
    title: 'The amount I payment',
    dataIndex: 'myAmount',
    key: 'amount'
  },
  {
    title: 'End time',
    dataIndex: 'endTime',
    key: 'endTime',
    slots: { customRender: 'time' }
  },
  {
    title: 'Current state',
    dataIndex: 'success',
    key: 'success',
    slots: { customRender: 'tag' }
  },
  {
    title: 'details',
    dataIndex: 'action',
    key: 'action',
    slots: { customRender: 'action' }
  },
]

export default defineComponent({
  name: 'Myself',
  components: { Modal, CreateForm, CheckCircleOutlined, SyncOutlined, CloseCircleOutlined },
  setup() {
    const isOpen = ref<boolean>(false);
    const state = reactive<{loading: boolean, init: Funding[], contr: Funding[]}>({
      loading: true,
      init: [],
      contr: []
    })

    async function fetchData() {
      state.loading = true;
      try {
        const res = await getMyFundings();
        console.log(res)
        state.init = res.init
        state.contr = res.contr
        state.loading = false;
      } catch (e) {
        console.log(e);
        message.error('Failed to Sell!');
      }
    }

    const router = useRouter();
    const clickFunding = (index : number) => {
      router.push(`/funding/${index}`)
    }
    addListener(fetchData)
    fetchData();

    return { state, columns, clickFunding }
  }
});
</script>
