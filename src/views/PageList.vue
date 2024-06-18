<script setup lang="ts">
  import { ref, onMounted } from 'vue'
  import {
    getFirestore,
    collection,
    query,
    orderBy,
    getDocs,
    deleteDoc,
    doc,
    where
  } from 'firebase/firestore'
  import { useUserStore } from '@/stores/user'
  import type { IInterview } from '@/interfaces'
  import { useConfirm } from 'primevue/useconfirm'

  const userStore = useUserStore()
  const db = getFirestore()
  const confirm = useConfirm()

  const interviews = ref<IInterview[]>([])
  const isLoading = ref<boolean>(true)
  const selectedFilterResult = ref<string | null>(null)
  const searchQuery = ref<string>('')

  const getAllInterviews = async <T extends IInterview>(
    isFilter?: boolean
  ): Promise<T[]> => {
    let getData
    if (isFilter) {
      getData = query(
        collection(db, `users/${userStore.userId}/interviews`),
        orderBy('createdAt', 'desc'),
        where('result', '==', selectedFilterResult.value)
      )
    } else {
      getData = query(
        collection(db, `users/${userStore.userId}/interviews`),
        orderBy('createdAt', 'desc')
      )
    }

    const listDocs = await getDocs(getData)

    return listDocs.docs.map(doc => doc.data() as T)
  }
  // Функция для поиска по названию компании
  const searchByCompanyName = () => {
    if (searchQuery.value.trim() === '') {
      // Если строка поиска пуста, получить все интервью
      getAllInterviews().then(interviewsList => {
        interviews.value = interviewsList
      })
    } else {
      // Фильтрация интервью по названию компании
      const filteredInterviews = interviews.value.filter(interview =>
        interview.company
          .toLowerCase()
          .includes(searchQuery.value.toLowerCase())
      )
      interviews.value = filteredInterviews
    }
  }

  const confirmRemoveInterview = async (id: string): Promise<void> => {
    confirm.require({
      message: 'Вы хотите удалить собеседование?',
      header: 'Удаление собеседований',
      icon: 'pi pi-info-circle',
      rejectLabel: 'Отмена',
      acceptLabel: 'Удалить',
      rejectClass: 'p-button-secondary p-button-outlined',
      acceptClass: 'p-button-danger',
      accept: async () => {
        isLoading.value = true
        await deleteDoc(doc(db, `users/${userStore.userId}/interviews`, id))

        const listIntervies: Array<IInterview> = await getAllInterviews()
        interviews.value = [...listIntervies]

        isLoading.value = false
      }
    })
  }

  const submitFilter = async (): Promise<void> => {
    isLoading.value = true
    const listIntervies: Array<IInterview> = await getAllInterviews(true)
    interviews.value = listIntervies
    isLoading.value = false
  }
  const clearFilter = async (): Promise<void> => {
    isLoading.value = true
    const listIntervies: Array<IInterview> = await getAllInterviews(false)
    selectedFilterResult.value = null // сбрасываем выбранную радиокнопку
    interviews.value = listIntervies
    isLoading.value = false
  }

  onMounted(async () => {
    const listIntervies: Array<IInterview> = await getAllInterviews()

    interviews.value = listIntervies

    isLoading.value = false
  })
</script>

<template>
  <app-dialog />
  <app-progress v-if="isLoading" />
  <div
    v-else-if="!isLoading && !interviews.length"
    class="flex align-items-center mr-2"
  >
    <app-message severity="info"> Нет добавленных собеседований </app-message>
    <!-- Сортировка по названию компании  -->
    <app-floatLabel class="flex align-items-center ml-2">
      <app-input-text
        id="companyName"
        v-model="searchQuery"
        @input="searchByCompanyName"
      />
      <label for="companyName">Компания</label>
    </app-floatLabel>

    <!--  -->
    <app-button
      class="ml-2"
      severity="danger"
      :disabled="!selectedFilterResult"
      @click="clearFilter"
      >Отменить</app-button
    >
  </div>
  <div v-else>
    <h1>Список заказов</h1>
    <div class="flex align-items-center mb-5">
      <div class="flex align-items-center mr-2">
        <app-radio
          inputId="interviewResult1"
          name="result"
          value="Refusal"
          v-model="selectedFilterResult"
        />
        <label for="interviewResult1" class="ml-2">Отказ</label>
      </div>
      <div class="flex align-items-center mr-2">
        <app-radio
          inputId="interviewResult2"
          name="result"
          value="Offer"
          v-model="selectedFilterResult"
        />
        <label for="interviewResult2" class="ml-2">Принято</label>
      </div>
      <app-button
        class="mr-2"
        @click="submitFilter"
        :disabled="!selectedFilterResult"
        >Применить</app-button
      >
      <app-button
        severity="danger"
        :disabled="!selectedFilterResult"
        @click="clearFilter"
        >Отменить</app-button
      >
      <!-- Сортировка по названию компании  -->
      <app-floatLabel>
        <app-input-text
          id="companyName"
          v-model="searchQuery"
          @input="searchByCompanyName"
          class="flex align-items-center ml-2"
        />
        <label class="ml-2" for="companyName">Компания</label>
      </app-floatLabel>
      <!--  -->
    </div>
    <app-datatable :value="interviews">
      <app-column field="company" header="Компания"></app-column>
      <app-column field="customersName" header="Имя заказчика"></app-column>
      <app-column field="vacancyLink" header="Вакансия">
        <template #body="slotProps">
          <a :href="slotProps.data.vacancyLink" target="_blank"
            >Ссылка на вакакнсию</a
          >
        </template>
      </app-column>
      <app-column header="Контакты">
        <template #body="slotProps">
          <div class="contacts">
            <a
              v-if="slotProps.data.contactTelegram"
              :href="`https://t.me/${slotProps.data.contactTelegram}`"
              target="_blank"
              class="contacts__telegram"
            >
              <span class="contacts__icon pi pi-telegram"></span>
            </a>
            <a
              v-if="slotProps.data.contactWhatsApp"
              :href="`https://wa.me/${slotProps.data.contactWhatsApp}`"
              target="_blank"
              class="contacts__whatsapp"
            >
              <span class="contacts__icon pi pi-whatsapp"></span>
            </a>
            <a
              v-if="slotProps.data.contactPhone"
              :href="`https://tel:${slotProps.data.contactPhone}`"
              target="_blank"
              class="contacts__phone"
            >
              <span class="contacts__icon pi pi-phone"></span>
            </a>
          </div>
        </template>
      </app-column>
      <app-column header="Пройденные этапы">
        <template #body="slotProps">
          <span v-if="!slotProps.data.stages">Не заполнено</span>
          <div v-else class="interview-stages">
            <app-badge
              v-for="(stage, i) in slotProps.data.stages"
              :key="i"
              :value="i + 1"
              rounded
              v-tooltip.top="stage.name"
            />
          </div>
        </template>
      </app-column>
      <app-column header="Ценовая вилка">
        <template #body="slotProps">
          <span v-if="!slotProps.data.salaryFrom"> Не заполнено </span>
          <span v-else>
            {{ slotProps.data.salaryFrom }} - {{ slotProps.data.salaryTo }}
          </span>
        </template>
      </app-column>
      <app-column header="Результат">
        <template #body="slotProps">
          <span v-if="!slotProps.data.result"> Не заполнено </span>
          <template v-else>
            <app-badge
              :severity="
                slotProps.data.result === 'Offer' ? 'success' : 'danger'
              "
              :value="slotProps.data.result === 'Offer' ? 'Принято' : 'Отказ'"
            ></app-badge>
          </template>
        </template>
      </app-column>
      <app-column>
        <template #body="slotProps">
          <div class="flex gap-2">
            <router-link :to="`/interview/${slotProps.data.id}`">
              <app-button icon="pi pi-pencil" severity="info" />
            </router-link>
            <app-button
              icon="pi pi-trash"
              severity="danger"
              @click="confirmRemoveInterview(slotProps.data.id)"
            />
          </div>
        </template>
      </app-column>
    </app-datatable>
  </div>
</template>

<style scoped>
  .contacts {
    display: flex;
    align-items: center;
    gap: 20px;
  }
  .contacts__telegram {
    color: #0088cc;
  }
  .contacts__whatsapp {
    color: #25d366;
  }
  .contacts__phone {
    color: #371777;
  }
  .contacts__icon {
    font-size: 20px;
  }
  .interview-stages {
    display: flex;
    gap: 5px;
  }
</style>
