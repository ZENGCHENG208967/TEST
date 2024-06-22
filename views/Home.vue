<template>
  <div>
    <a-card class="ant-card-shadow">
      <template #title>
        <h3>
          All Motor transaction Project
          <a-button style="float: right" @click="openModal" type="primary">Launch Project</a-button>
		  <a-select v-model:value="filterType" style="width: 250px" placeholder="Classification" :options="options" @change="handleChange"></a-select>
		 </h3>
      </template>
      <a-table :columns="columns" :loading="state.loading" :data-source="state.data">
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
            Purchase
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

    <Modal v-model:visible="isOpen">
      <a-card style="width: 600px; margin: 0 2em;" :body-style="{ overflowY: 'auto', maxHeight: '600px' }">
        <template #title><h3 style="text-align: center">Launch Project</h3></template>
        <create-form :model="model" :form="form" :fields="fields" />
      </a-card>
    </Modal>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, reactive } from 'vue';
import Modal from '../components/base/modal.vue'
import CreateForm from '../components/base/createForm.vue'
import { Model, Fields, Form } from '../type/form'
import { contract, getAccount, getAllFundings, Funding, newFunding, addListener } from '../api/contract'
import { message } from 'ant-design-vue'
import { CheckCircleOutlined, SyncOutlined, CloseCircleOutlined } from '@ant-design/icons-vue'
import { useRouter } from 'vue-router'

const columns = [
  {
    dataIndex: 'title',
    key: 'title',
    title: 'Project title'
  },
  {
    title: 'Target amount(eth)',
    dataIndex: 'goal',
    key: 'goal'
  },
  {
    title: 'Type',
    dataIndex: 'status',
    key: 'status'
  },
  {
    title: 'Current amount(eth)',
    dataIndex: 'amount',
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
  }
]

export default defineComponent({
  name: 'Home',
  components: { Modal, CreateForm, CheckCircleOutlined, SyncOutlined, CloseCircleOutlined },
  setup() {
    const isOpen = ref<boolean>(false);
    const state = reactive<{loading: boolean, data: Funding[]}>({
      loading: true,
      data: []
    })

	const filterType = ref<string|undefined>(undefined);
    async function fetchData() {
      state.loading = true;
      try {
        state.data = await getAllFundings(filterType.value);
        state.loading = false;
      } catch (e) {
        console.log(e);
        message.error('Get Purchase failure!');
      }
    }

    async function openModal() { 
      model.account = await getAccount();
      isOpen.value = true;
    }
    function closeModal() { isOpen.value = false; }

    const model = reactive<Model>({
      account: '',
      title: '',
      info: '',
      amount: 0,
      date: null,
    })

    const fields = reactive<Fields>({
      account: {
        type: 'input',
        label: 'Seller',
        disabled: true
      },
      title: {
        type: 'input',
        label: 'title',
        rule: {
          required: true,
          trigger: 'blur'
        }
      },
      info: {
        type: 'textarea',
        label: 'intro',
        rule: {
          required: true,
          trigger: 'blur'
        }
      },
      amount: {
        type: 'number',
        label: 'amount',
        min: 0
      },
      date: {
        type: 'time',
        label: 'Deadline date',
      },
	  status: {
		  type: 'select',
		  label: 'status',
		  select: ['new','used']
	  },
	  imgs: {
		  type: 'upload',
		  label: 'imgs'
	  }
    })

    const form = reactive<Form>({
      submitHint: 'Launch Project',
      cancelHint: 'Cancel',
      cancel: () => {
        closeModal();
      },
      finish: async () => {
        const seconds = Math.ceil(new Date(model.date).getTime() / 1000);
        try {
		  console.log(model)
          const res = await newFunding(model.account, model.title, model.info, model.amount, seconds, model.status, model.imgs);
          console.log(res)
          message.success('Successfully Created Project')
          closeModal();
          fetchData();
        } catch(e) {
          console.log(e);
          message.error('Purchase failed')
        }
      }
    })

    const router = useRouter();
    const clickFunding = (index : number) => {
      router.push(`/funding/${index}`)
    }
    addListener(fetchData)
    fetchData();
	
	const handleChange = (value: string) => {
	  filterType.value = value;
	  fetchData();
	};
	
    return { 
		openModal, 
		isOpen, 
		model, 
		fields, 
		form, 
		state, 
		columns, 
		clickFunding ,
		filterType,
		handleChange,
		options: ['new','used'].map((_, i) => ({ value: _ })),
	}
  }
});
</script>
