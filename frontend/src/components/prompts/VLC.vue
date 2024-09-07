<template>
  <div class="card floating" id="quick-link">
    <div class="card-title">
      <h2>VLC</h2>
    </div>

    <template v-if="listing">
      <div class="card-content">
      </div>

      <div class="card-action">
        <button
            id="focus-prompt"
            class="button button--flat button--grey"
            @click="() => {
            if (this.links.length === 0) {
              submit();
            }

            if (this.links.length > 0) {
              copyToClipboard(buildDownloadLink(this.links[this.links.length - 1]));
            }

            this.closeHovers();
          }"
            :aria-label="$t('buttons.close')"
            :title="$t('buttons.close')"
            tabindex="2"
        >
          Copy
        </button>
      </div>
    </template>
  </div>
</template>

<script>
import { mapActions, mapState } from "pinia";
import { useFileStore } from "@/stores/file";
import { share as api, pub as pub_api } from "@/api";
import { useLayoutStore } from "@/stores/layout";
import { copy } from "@/utils/clipboard";

export default {
  name: "quick-link",
  data: function () {
    return {
      time: 0,
      unit: "hours",
      links: [],
      clip: null,
      password: "",
      listing: true,
    };
  },
  inject: ["$showError", "$showSuccess"],
  computed: {
    ...mapState(useFileStore, [
      "req",
      "selected",
      "selectedCount",
      "isListing",
    ]),
    url() {
      if (!this.isListing) {
        return this.$route.path;
      }

      if (this.selectedCount === 0 || this.selectedCount > 1) {
        // This shouldn't happen.
        return;
      }

      return this.req.items[this.selected[0]].url;
    },
  },
  async beforeMount() {
    try {
      const links = await api.get(this.url);
      this.links = links;
      this.sort();

      if (this.links.length == 0) {
        this.listing = false;
      }
    } catch (e) {
      this.$showError(e);
    }
  },
  async mounted() {
    const links = await api.get(this.url);

    if (links.length === 0) {
      await this.submit();
    }

    if (links.length > 0) {
      const link = this.buildDownloadLink(links[links.length - 1]);
      this.copyToClipboard(link);

      this.openVLC(link);
    }

    this.closeHovers();
  },
  methods: {
    ...mapActions(useLayoutStore, ["closeHovers"]),
    openVLC(url) {
      window.location.href = `vlc://${url}`;
    },
    copyToClipboard: function (text) {
      copy(text).then(
          () => {
            // clipboard successfully set
            this.$showSuccess(this.$t("success.linkCopied"));
          },
          () => {
            // clipboard write failed
          }
      );
    },
    submit: async function () {
      try {
        let res;

        if (!this.time) {
          res = await api.create(this.url, this.password);
        } else {
          res = await api.create(this.url, this.password, this.time, this.unit);
        }

        this.links.push(res);
        this.sort();

        this.time = 0;
        this.unit = "hours";
        this.password = "";

        this.listing = true;
      } catch (e) {
        this.$showError(e);
      }
    },
    buildDownloadLink(share) {
      return pub_api.getDownloadURL(share);
    },
    sort() {
      this.links = this.links.sort((a, b) => {
        if (a.expire === 0) return -1;
        if (b.expire === 0) return 1;
        return new Date(a.expire) - new Date(b.expire);
      });
    },
  },
};
</script>
