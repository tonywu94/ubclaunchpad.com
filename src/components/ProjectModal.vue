<template>
  <div
    class="modal"
    :class="{
      'is-active': isActive,
    }"
  >
    <div
      class="modal-background animated fadeIn faster"
      @click="handleModalClose()"
    />

    <div class="modal-content box-shadow animated zoomIn faster">
      <unicon
        name="times"
        class="close-button icon-small hoverable"
        @click="handleModalClose()"
      />
      <unicon
        id="share-button"
        name="share-alt"
        class="share-button icon-small hoverable"
        @click="shareToClipboard()"
      />

      <div class="pad-sides-8 has-text-centered">
        <h2 class="accent">
          {{ project.name }}
        </h2>
        <h3 class="margin-sides-16">
          {{ project.description }}
        </h3>

        <div
          v-if="isActive"
          class="media-container margin-sides-16"
        >
          <img
            v-if="!project.media"
            :src="project.banner.url"
            :alt="project.name"
          >
          <img
            v-else-if="project.media.type == 'image'"
            :src="project.media.url"
            :alt="project.name"
          >
          <div
            v-else-if="project.media.type == 'youtube'"
            class="video-container"
          >
            <iframe
              id="ytplayer"
              ref="ytplayer-iframe"
              :src="generateYouTubeEmbedSrc(project.media)"
              type="text/html"
              class="video-player"
              allowFullScreen
              frameborder="0"
            />
          </div>
        </div>

        <p
          v-if="isActive"
          class="socials"
        >
          <a
            data-tippy-content="GitHub"
            :href="project.links.repository"
            target="_blank"
            rel="noopener noreferrer"
            @click="onLinkClick"
          >
            <unicon
              name="github-alt"
              class="icon-small hoverable"
            />
          </a>
          <a
            v-if="project.links.website"
            data-tippy-content="Website"
            :href="project.links.website"
            target="_blank"
            rel="noopener noreferrer"
            @click="onLinkClick"
          >
            <unicon
              name="window"
              class="icon-small hoverable"
            />
          </a>
          <a
            v-if="project.links.app && project.links.app.type === 'ios'"
            data-tippy-content="App Store"
            :href="project.links.app.url"
            target="_blank"
            rel="noopener noreferrer"
            @click="onLinkClick"
          >
            <unicon
              name="apple-alt"
              class="icon-small hoverable"
            />
          </a>
          <a
            v-if="project.links.app && project.links.app.type === 'android'"
            data-tippy-content="Play Store"
            :href="project.links.app.url"
            target="_blank"
            rel="noopener noreferrer"
            @click="onLinkClick"
          >
            <unicon
              name="android-alt"
              class="icon-small hoverable"
            />
          </a>
          <a
            v-if="project.links.writeup"
            data-tippy-content="Writeup"
            :href="project.links.writeup"
            target="_blank"
            rel="noopener noreferrer"
            @click="onLinkClick"
          >
            <unicon
              name="notebooks"
              class="icon-small hoverable"
            />
          </a>
        </p>

        <p
          v-if="project.elevatorPitch"
          class="margin-sides-16"
        >
          {{ project.elevatorPitch }}
        </p>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue';
import tippy from 'tippy.js';
import { Project, MediaYouTube } from '@/data/types';

// see https://github.com/ubclaunchpad/ubclaunchpad.com/issues/105
import * as clipboard from 'clipboard-polyfill';

export default Vue.extend({
  name: 'ProjectModal',
  props: {
    /**
     * Name of section this modal is based in
     */
    section: { type: String, required: true },
    /**
     * Project to display
     */
    project: {
      type: Object as () => Project,
      required: true,
    },
    /**
     * Toggle whether modal is active
     */
    isActive: Boolean,
  },
  updated() {
    // only attach tooltips after entire view has been rendered
    if (this.isActive) {
      this.$nextTick(function() { tippy('[data-tippy-content]'); });
    }
  },
  created() {
    // check for tooltips on first mount as well - modal is only ever active on mount if 
    if (this.isActive) { tippy('[data-tippy-content]'); }
  },
  methods: {
    /**
     * Reports an analytics event
     */
    reportEvent(action: string, project: Project) {
      this.$gtag.event(action, {
        event_category: this.$options.name,
        // include which section this modal interaction came from
        event_label: `${this.section}: ${project.name}`,
      });
    },
    /**
     * Closes this modal
     */
    handleModalClose() {
      this.$emit('modalClosed');
    },
    /**
     * Shares this project modal to clipboard, and show a tooltip indicating success
     */
    async shareToClipboard() {
      this.reportEvent('project-modal-share', this.project);

      // encode share link and write to clipboard
      const urlParams = new URLSearchParams({
        project: this.project.name.toLowerCase(),
      } as Record<string, string>);
      const { protocol, host } = window.location;
      await clipboard.writeText(`${protocol}//${host}?${urlParams.toString()}#${this.section.toLowerCase()}`);

      // show and destroy tooltip after a few seconds
      const tooltip = tippy('#share-button', {
        content: 'Link copied to clipboard!',
        trigger: 'manual',
        placement: 'top-end',
      })[0];
      tooltip.show();
      setTimeout(() => tooltip.destroy(), 3000);
    },
    /**
     * Generate a YouTube link specifically for use with our iframe embed.
     */
    generateYouTubeEmbedSrc(media: MediaYouTube): string {
      // see https://developers.google.com/youtube/player_parameters#Parameters
      const params: Record<string, string> = {
        frameborder: '0', // no ugly border
        modestbranding: '1', // reduce youtube branding
        rel: '0', // no related videos
        origin: window.location.host,
      };
      if (media.startAt) params.start = `${media.startAt}`;
      const urlParams = new URLSearchParams(params);
      return `https://youtube.com/embed/${media.id}?${urlParams.toString()}`;
    },
    /**
     * Report a link click - for now, don't bother checking which exact link, since any click is
     * probably an exciting event.
     */
    onLinkClick() {
      this.reportEvent('project-modal-link-click', this.project);
    },
  },
});
</script>

<style scoped lang="scss">
.modal-background {
  background-color: rgba($black, 0.9);
}

.modal-content {
  border-radius: 15px;
  background-color: $dark;
  box-shadow: 0 0 50px rgba($accent, 0.3);
  max-height: calc(100vh - 24px);

  * {
    position: relative;
  }

  .close-button {
    z-index: 99;
    cursor: pointer;
  }

  .share-button {
    z-index: 99;
    cursor: pointer;
    float: right;
    margin-right: 8px;
  }

  h2 {
    margin-bottom: 8px;
    @media (max-width: $tablet) {
      font-size: 24px;
    }
  }

  h3 {
    margin-bottom: 16px;
  }

  .socials {
    margin-bottom: 0px;

    a {
      margin-right: 12px;
      margin-left: 12px;
    }
  }

  .media-container {
    margin-bottom: 8px;

    img {
      border-radius: 8px;
    }

    // https://css-tricks.com/fluid-width-video/#article-header-id-0
    .video-container {
      position: relative;
      padding-bottom: 56.25%; /* 16:9 */
      height: 0;
      .video-player {
        border-radius: 8px;
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
      }
    }
  }
}
</style>
