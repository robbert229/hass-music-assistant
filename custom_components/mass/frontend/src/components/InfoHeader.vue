<template>
  <div>
    <v-card
      flat
      :img="imgGradient"
      style="margin-top: -10px; z-index: 0"
      min-height="150"
    >
      <v-img
        width="100%"
        height="100%"
        cover
        class="background-image"
        :src="fanartImage"
        :gradient="
          store.darkTheme
            ? 'to bottom, rgba(0,0,0,.90), rgba(0,0,0,.75)'
            : 'to bottom, rgba(255,255,255,.90), rgba(255,255,255,.75)'
        "
      >
      </v-img>
      <v-layout v-if="item" style="padding-left: 15px; padding-right: 15px">
        <!-- left side: cover image -->
        <div v-if="!$vuetify.display.mobile" xs5 pa-5>
          <MediaItemThumb
            :item="item"
            :size="180"
            width="100%"
            :min-width="180"
            :min-height="180"
            :border="true"
            :fallback="imgDefaultArtist"
            style="margin-top: 15px; margin-bottom: 15px"
          />
        </div>

        <div>
          <!-- Main title -->
          <v-card-title>
            {{ item.name }}
          </v-card-title>

          <!-- other details -->
          <div style="padding-bottom: 10px">
            <!-- version -->
            <v-card-subtitle
              v-if="'version' in item && item.version"
              class="caption"
            >
              {{ item.version }}
            </v-card-subtitle>

            <!-- item artists -->
            <v-card-subtitle
              v-if="'artists' in item && item.artists"
              class="title accent--text"
            >
              <v-icon
                style="margin-left: -3px; margin-right: 3px"
                small
                color="primary"
                :icon="mdiAccountMusic"
              ></v-icon>
              <span
                v-for="(artist, artistindex) in item.artists"
                :key="artist.item_id"
              >
                <a style="color: accent" @click="artistClick(artist)">{{
                  artist.name
                }}</a>
                <span
                  v-if="artistindex + 1 < item.artists.length"
                  :key="artistindex"
                  style="color: accent"
                  >{{ " / " }}</span
                >
              </span>
            </v-card-subtitle>

            <!-- album artist -->
            <v-card-subtitle
              v-if="'artist' in item && item.artist"
              class="title"
            >
              <v-icon
                style="margin-left: -3px; margin-right: 3px"
                small
                color="primary"
                :icon="mdiAccountMusic"
              ></v-icon>
              <a @click="artistClick(item.artist)">{{ item.artist.name }}</a>
            </v-card-subtitle>

            <!-- playlist owner -->
            <v-card-subtitle v-if="'owner' in item && item.owner" class="title">
              <v-icon
                color="primary"
                style="margin-left: -3px; margin-right: 3px"
                small
                :icon="mdiAccountMusic"
              ></v-icon>
              <a style="color: primary">{{ item.owner }}</a>
            </v-card-subtitle>

            <v-card-subtitle v-if="'album' in item && item.album">
              <v-icon
                color="primary"
                style="margin-left: -3px; margin-right: 3px"
                small
                :icon="mdiAlbum"
              ></v-icon>
              <a style="color: secondary" @click="albumClick(item.album)">{{
                item.album.name
              }}</a>
            </v-card-subtitle>
          </div>

          <!-- play/info buttons -->
          <div style="margin-left: 14px; padding-bottom: 10px">
            <v-btn
              color="primary"
              tile
              :prepend-icon="mdiPlayCircle"
              @click="
                store.contextMenuItems = [item];
                store.showContextMenu = true;
              "
            >
              {{ $t("play") }}
            </v-btn>
            <v-btn
              v-if="!$vuetify.display.mobile && !item.in_library"
              style="margin-left: 10px"
              color="primary"
              tile
              :prepend-icon="mdiHeartOutline"
              @click="api.addToLibrary([item])"
            >
              {{ $t("add_library") }}
            </v-btn>
            <v-btn
              v-if="!$vuetify.display.mobile && item.in_library"
              style="margin-left: 10px"
              color="primary"
              tile
              :prepend-icon="mdiHeart"
              @click="api.removeFromLibrary([item])"
            >
              {{ $t("remove_library") }}
            </v-btn>
          </div>

          <!-- Description/metadata -->
          <v-card-subtitle
            class="body-2 justify-left"
            style="padding-bottom: 10px"
            @click="showFullInfo = !showFullInfo"
            v-if="description"
          >
            <span>
              <div v-html="description"> </div>
              <v-btn
                variant="plain"
                :icon="showFullInfo ? mdiChevronUp : mdiChevronDown"
                style="padding: 0; height: 20px"
              ></v-btn>
            </span>
            </v-card-subtitle>


          <!-- genres/tags -->
          <div
            v-if="item && item.metadata.genres"
            class="justify-center"
            style="margin-left: 15px; padding-bottom: 20px"
          >
            <v-chip
              v-for="tag of item.metadata.genres"
              :key="tag"
              color="blue-grey lighten-1"
              style="margin-right: 5px; margin-bottom: 5px"
              small
              outlined
              >{{ tag }}</v-chip
            >
          </div>
        </div>
        <!-- provider icons -->
        <div style="position: absolute; float: right; right: 15px; top: 15px">
          <ProviderIcons
            :provider-ids="item.provider_ids"
            :height="25"
            :enable-link="true"
          />
        </div>
      </v-layout>
    </v-card>
  </div>
</template>

<script setup lang="ts">
import ProviderIcons from "./ProviderIcons.vue";
import {
  mdiChevronDown,
  mdiChevronUp,
  mdiHeart,
  mdiHeartOutline,
  mdiAccountMusic,
  mdiAlbum,
  mdiPlayCircle,
} from "@mdi/js";

import { store } from "../plugins/store";
import { useDisplay } from "vuetify";
import api from "../plugins/api";
import { ImageType } from "../plugins/api";
import type { Album, Artist, ItemMapping, MediaItemType } from "../plugins/api";
import { computed, onBeforeUnmount, ref, watchEffect } from "vue";
import MediaItemThumb from "./MediaItemThumb.vue";
import { getImageThumbForItem } from "./MediaItemThumb.vue";
import { useI18n } from "vue-i18n";
import { useRouter } from "vue-router";
import { truncateString } from "@/utils";

// properties
export interface Props {
  item?: MediaItemType;
}
const props = defineProps<Props>();
const showFullInfo = ref(false);
const fanartImage = ref();
const display = useDisplay();

const imgGradient = new URL("../assets/info_gradient.jpg", import.meta.url)
  .href;
const imgDefaultArtist = new URL(
  "../assets/default_artist.png",
  import.meta.url
).href;

const { t } = useI18n();
const router = useRouter();

watchEffect(async () => {
  if (props.item) {
    if (display.mobile) {
      store.topBarTitle = truncateString(props.item.name, 30);
    } else {
      store.topBarTitle =
        '<span style="opacity:0.5">' +
        t(props.item.media_type + "s") +
        ` | </span>${props.item.name}`;
    }

    store.contextMenuParentItem = props.item;
    fanartImage.value =
      (await getImageThumbForItem(props.item, ImageType.FANART)) ||
      (await getImageThumbForItem(props.item, ImageType.THUMB));
  }
});

onBeforeUnmount(() => {
  store.contextMenuParentItem = undefined;
});

const albumClick = function (item: Album | ItemMapping) {
  // album entry clicked
  router.push({
    name: "album",
    params: {
      item_id: item.item_id,
      provider: item.provider,
    },
  });
};
const artistClick = function (item: Artist | ItemMapping) {
  // album entry clicked
  router.push({
    name: "artist",
    params: {
      item_id: item.item_id,
      provider: item.provider,
    },
  });
};
const description = computed(() => {
  let desc = "";
  if (!props.item) return "";
  if (props.item.metadata && props.item.metadata.description) {
    desc = props.item.metadata.description;
  } else if (props.item.metadata && props.item.metadata.copyright) {
    desc = props.item.metadata.copyright;
  } else if ("artists" in props.item) {
    props.item.artists.forEach(function (artist: Artist | ItemMapping) {
      if ("metadata" in artist && artist.metadata.description) {
        desc = artist.metadata.description;
      }
    });
  }
  const maxChars = display.mobile ? 160 : 380;
  desc = desc.replace("\r\n", "<br /><br /><br />");
  desc = desc.replace("\r", "<br /><br />");
  desc = desc.replace("\n", "<br /><br />");
  if (showFullInfo.value) return desc;
  if (desc.length > maxChars) {
    return desc.substring(0, maxChars) + "...";
  }
  return desc;
});
</script>

<style>
.background-image {
  position: absolute;
}
.background-image .v-img__img--cover {
  object-position: 50% 20%;
}
</style>
