<template>
  <section>
    <InfoHeader :item="track" />
    <v-tabs v-model="activeTab" show-arrows grow hide-slider>
      <v-tab
        :class="activeTab == 'versions' ? 'active-tab' : 'inactive-tab'"
        value="versions"
      >
        {{ $t("track_versions") }} ({{ trackVersions.length }})</v-tab
      >
      <v-tab
        :class="activeTab == 'details' ? 'active-tab' : 'inactive-tab'"
        value="details"
      >
        {{ $t("details") }}</v-tab
      >
    </v-tabs>
    <v-divider />
    <ItemsListing
      :items="trackVersions"
      itemtype="tracks"
      :loading="loading"
      :parent-item="track"
      :show-providers="true"
      v-if="activeTab == 'versions'"
    />
    <div v-if="activeTab == 'details'">
      <v-table style="width: 100%">
        <thead>
          <tr>
            <th class="text-left">Provider</th>
            <th class="text-left">ID</th>
            <th class="text-left">Available ?</th>
            <th class="text-left">Quality</th>
            <th class="text-left">details</th>
            <th class="text-left">preview</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) of track?.provider_ids" :key="item.item_id">
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
            <td
              class="details-column"
              @mouseover="fetchPreviewUrl(item.prov_type, item.item_id, index)"
            >
              <audio
                style="width: 260px"
                controls
                :src="previewUrls[index]"
              ></audio>
            </td>
          </tr>
        </tbody>
      </v-table>
    </div>
  </section>
</template>

<script setup lang="ts">
import ItemsListing from "../components/ItemsListing.vue";
import InfoHeader from "../components/InfoHeader.vue";
import { ref, reactive } from "@vue/reactivity";
import { useTheme } from "vuetify";
import type { MassEvent, ProviderType, Track } from "../plugins/api";
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
const activeTab = ref("versions");

const track = ref<Track>();
const trackVersions = ref<Track[]>([]);
const loading = ref(true);
const previewUrls = reactive<Record<number, string>>({});
const theme = useTheme();

watchEffect(async () => {
  api
    .getTrack(
      props.provider as ProviderType,
      props.item_id,
      parseBool(props.lazy),
      parseBool(props.refresh)
    )
    .then(async (item) => {
      track.value = item;
      // fetch additional info once main info retrieved
      trackVersions.value = await api.getTrackVersions(
        props.provider as ProviderType,
        props.item_id
      );
      loading.value = false;
    });
});

const fetchPreviewUrl = async function (
  provider: ProviderType,
  item_id: string,
  index: number
) {
  if (index in previewUrls) return;
  const url = await api.getTrackPreviewUrl(provider, item_id);
  previewUrls[index] = url;
};

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
    track.value = newItem;
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
