<script setup lang="ts">
  import { ref, computed } from 'vue'
  import { useRouter } from 'vue-router'
  import { getFirestore, setDoc, doc } from 'firebase/firestore'
  import type { IInterview } from '@/interfaces'
  import { v4 as uuidv4 } from 'uuid'
  import { useUserStore } from '@/stores/user'

  const userStore = useUserStore()
  const db = getFirestore()
  const router = useRouter()

  const company = ref<string>('')
  const vacancyLink = ref('')
  const customersName = ref<string>('')
  const contactTelegram = ref<string>('')
  const contactWhatsApp = ref<string>('')
  const contactPhone = ref<string>('')

  const loading = ref<boolean>(false)

  const addNewInterview = async (): Promise<void> => {
    loading.value = true
    const payload: IInterview = {
      id: uuidv4(),
      company: company.value,
      vacancyLink: vacancyLink.value,
      customersName: customersName.value,
      contactTelegram: contactTelegram.value,
      contactWhatsApp: contactWhatsApp.value,
      contactPhone: contactPhone.value,
      createdAt: new Date()
    }

    if (userStore.userId) {
      await setDoc(
        doc(db, `users/${userStore.userId}/interviews`, payload.id),
        payload
      ).then(() => {
        router.push('/list')
      })
    }
  }

  const disabledSaveButton = computed<boolean>(() => {
    return !(company.value && vacancyLink.value && customersName.value)
  })
</script>

<template>
  <div class="content-interview">
    <app-card>
      <template #title>Новый заказ на разработку</template>
      <template #content>
        <app-input-text
          class="input mb-3"
          placeholder="Компания"
          v-model="company"
        />
        <app-input-text
          v-model="vacancyLink"
          class="input mb-3"
          placeholder="Описание заказа (ссылка)"
        />
        <app-input-text
          v-model="customersName"
          class="input mb-3"
          placeholder="Имя заказчика"
        />
        <app-input-text
          v-model="contactTelegram"
          class="input mb-3"
          placeholder="Telegram username "
        />
        <app-input-text
          v-model="contactWhatsApp"
          class="input mb-3"
          placeholder="WhatsApp телефон "
        />
        <app-input-text
          v-model="contactPhone"
          class="input mb-3"
          placeholder="Телефон "
        />
        <app-button
          @click="addNewInterview"
          label="Создать собеседование"
          :disabled="disabledSaveButton"
          :loading="loading"
        ></app-button>
      </template>
    </app-card>
  </div>
</template>

<style scoped>
  .input {
    width: 100%;
  }
  .content-interview {
    max-width: 600px;
    margin: auto;
  }
</style>
