<template>
  <section>
    <InfoHeader :item="radio" />
    <v-tabs v-model="activeTab" show-arrows grow hide-slider>
      <v-tab
        :class="activeTab == 'details' ? 'active-tab' : 'inactive-tab'"
        value="details"
      >
        {{ $t("details") }}</v-tab
      >
    </v-tabs>
    <v-divider />
    <div v-if="activeTab == 'details'">
      <v-table style="width: 100%">
        <thead>
          <tr>
            <th class="text-left">Provider</th>
            <th class="text-left">ID</th>
            <th class="text-left">Available ?</th>
            <th class="text-left">Quality</th>
            <th class="text-left">details</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item of radio?.provider_ids" :key="item.item_id">
            <td class="details-column">
              <v-img
                width="25px"
                :src="getProviderIcon(item.prov_type)"
              ></v-img>
            </td>
            <td class="details-column">
              <a :href="item.url" target="_blank">{{ item.item_id }}</a>
            </td>
            <td class="details-column">{{ item.available }}</td>
            <td class="details-column">
              <v-img
                width="35px"
                :src="getQualityIcon(item.quality)"
                :style="
                  $vuetify.theme.current.dark
                    ? 'object-fit: contain;' 
                    : 'object-fit: contain;filter: invert(100%);'
                "
              ></v-img>
            </td>
            <td class="details-column">{{ item.details }}</td>
          </tr>
        </tbody>
      </v-table>
    </div>
  </section>
</template>

<script setup lang="ts">
import InfoHeader from "../components/InfoHeader.vue";
import { useTheme } from "vuetify";
import { ref } from "@vue/reactivity";
import type { MassEvent, ProviderType, Radio, Track } from "../plugins/api";
import { api, MassEventType } from "../plugins/api";
import {
  getProviderIcon,
  getQualityIcon,
} from "../components/ProviderIcons.vue";
import { onBeforeUnmount, watchEffect } from "vue";
import { parseBool } from "../utils";

export interface Props {
  item_id: string;
  provider: string;
  lazy?: boolean | string;
  refresh?: boolean | string;
}
const props = withDefaults(defineProps<Props>(), {
  lazy: true,
  refresh: false,
});
const activeTab = ref("details");

const radio = ref<Radio>();
const loading = ref(true);
const theme = useTheme();

watchEffect(async () => {
  const item = await api.getRadio(
    props.provider as ProviderType,
    props.item_id,
    parseBool(props.lazy),
    parseBool(props.refresh)
  );
  radio.value = item;
  loading.value = false;
});

// listen for item updates to refresh interface when that happens
const unsub = api.subscribe(MassEventType.MEDIA_ITEM_UPDATED, (evt: MassEvent) => {
  const newItem = evt.data as Track;
  if (
    (props.provider == "database" && newItem.item_id == props.item_id) ||
    newItem.provider_ids.filter(
      (x) => x.prov_type == props.provider && x.item_id == props.item_id
    ).length > 0
  ) {
    // got update for current item
    radio.value = newItem;
  }
});
onBeforeUnmount(unsub);
</script>

<style>
.details-column {
  max-width: 200px;
  width: 30px;
  overflow: hidden;
  text-overflow: ellipsis;
}
</style>
