<template lang='pug'>
  img.c-avatar(
    v-if='imageURL'
    :class='`is-${size}`'
    :src='imageURL'
    :alt='alt'
    ref='img'
    v-on='$listeners'
  )
  .c-avatar.is-empty(v-else :class='`is-${size}`')
</template>

<script>
import sbp from '~/shared/sbp.js'
import { handleFetchResult } from '~/frontend/controller/utils/misc.js'

export default {
  name: 'Avatar',
  props: {
    src: String, // acts as a placeholder when used together with blobURL
    alt: {
      type: String,
      default: ''
    },
    blobURL: String,
    size: {
      type: String,
      default: 'md',
      validator: (value) => ['xs', 'sm', 'md', 'lg', 'xl'].includes(value)
    }
  },
  mounted () {
    console.log(`Avatar under ${this.$parent.$vnode.tag} blobURL:`, this.blobURL, 'src:', this.src)
    if (this.blobURL && !this.objectURL) {
      // trigger the watcher
      this.setBlobURL(this.blobURL)
    }
  },
  data () {
    return {
      objectURL: ''
    }
  },
  methods: {
    setFromBlob (blob) {
      // free this resource per https://developer.mozilla.org/en-US/docs/Web/API/File/Using_files_from_web_applications#Using_object_URLs
      this.$refs.img.onload = function () { URL.revokeObjectURL(this.src) }
      this.objectURL = URL.createObjectURL(blob)
    },
    async setBlobURL (newBlobURL) {
      const oldBlobURL = this.blobURL
      if (!newBlobURL) {
        this.objectURL = ''
        return
      } // don't do anything if newBlobURL is blank
      try {
        const url = new URL(newBlobURL)
        const imageType = url.searchParams.get('type')
        let blob = await sbp('gi.db/files/load', newBlobURL)
        let b
        if (!blob) {
          console.debug('[DEBUG] Avatar: no blob loaded for', newBlobURL)
          // we need to retrieve it from the server and save it.
          // To protect privacy, we fetch using a URL that doesn't have the mime-type in it
          blob = await fetch(`${url.origin}${url.pathname}`)
            .then(handleFetchResult('blob'))
          // cache the image locally
          await sbp('gi.db/files/save', newBlobURL, blob)
        } else {
          console.debug('[DEBUG] Avatar: loaded blob for', newBlobURL, blob)
        }
        // the image is already cached
        if (!imageType) {
          console.warn('Avatar.vue: no mimetype!', newBlobURL)
          b = new Blob([blob])
        } else {
          b = new Blob([blob], { type: imageType })
        }
        if (oldBlobURL && oldBlobURL !== newBlobURL) {
          console.debug('[DEBUG] Avatar: deleting oldBlobURL from cache:', oldBlobURL)
          await sbp('gi.db/files/delete', oldBlobURL)
        }
        this.setFromBlob(b)
      } catch (e) {
        console.error('Avatar: error loading blobURL', e.stack || e)
      }
    }
  },
  watch: {
    blobURL (newBlobURL, oldBlobURL) {
      console.debug(`Avatar blobURL watcher: ${oldBlobURL} => ${newBlobURL}`)
      this.setBlobURL(newBlobURL)
    }
  },
  computed: {
    imageURL () {
      return this.objectURL || this.src
    }
  }
}
</script>

<style lang="scss" scoped>
@import "@assets/style/_variables.scss";

@mixin size($size) {
  width: $size;
  height: $size;
}

.c-avatar {
  border-radius: 50%;
  object-fit: cover;
  flex-shrink: 0;

  &::after {
    content: "";
    display: block;
    padding-bottom: 100%;
  }

  &.is-empty {
    background-color: $general_0;
  }

  &.is-xs { @include size(1.5rem); }
  &.is-sm { @include size(2rem); }
  &.is-md { @include size(2.5rem); }
  &.is-lg { @include size(4.5rem); }
  &.is-xl { @include size(8rem); }
}
</style>
