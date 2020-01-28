<template>
    <div class="modal fade" tabindex="-1" role="dialog" aria-hidden="true">
        <div class="modal-dialog" ref="modal" role="document">
            <div class="modal-content">
                <div v-if="!selectedImageUrl" class="modal-header d-flex align-items-center justify-content-between">
                    <div v-if="unsplashKey" class="input-group">
                        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="20" class="icon-search">
                            <circle cx="10" cy="10" r="7" class="fill-bg"/>
                            <path class="primary" d="M16.32 14.9l1.1 1.1c.4-.02.83.13 1.14.44l3 3a1.5 1.5 0 0 1-2.12 2.12l-3-3a1.5 1.5 0 0 1-.44-1.14l-1.1-1.1a8 8 0 1 1 1.41-1.41zM10 16a6 6 0 1 0 0-12 6 6 0 0 0 0 12z"/>
                        </svg>
                        <input
                            v-model="searchKeyword"
                            type="text"
                            autofocus
                            class="form-control border-0 bg-transparent"
                            :placeholder="trans.posts.forms.editor.images.picker.placeholder"
                        />
                    </div>

                    <button type="button" class="close" data-dismiss="modal" aria-label="Close" @click.prevent="closeModal">
                        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" class="icon-close-circle">
                            <circle cx="12" cy="12" r="10" class="primary"/>
                            <path class="fill-bg" d="M13.41 12l2.83 2.83a1 1 0 0 1-1.41 1.41L12 13.41l-2.83 2.83a1 1 0 1 1-1.41-1.41L10.59 12 7.76 9.17a1 1 0 0 1 1.41-1.41L12 10.59l2.83-2.83a1 1 0 0 1 1.41 1.41L13.41 12z"/>
                        </svg>
                    </button>
                </div>
                <div class="modal-body pb-0">
                    <file-pond
                        v-if="!isSearchingUnsplash && !unsplashImages.length && isReadyToAcceptUploads"
                        name="featuredImagePond"
                        ref="pond"
                        max-files="1"
                        :label-idle="getPlaceholderLabel"
                        accepted-file-types="image/jpeg, image/png"
                        :server="getServerOptions"
                        :allow-multiple="false"
                        :files="selectedImagesForPond"
                        @processfile="processedFromFilePond"
                        @removefile="removedFromFilePond"/>

                    <div v-if="unsplashKey && !selectedImageUrl">
                        <div v-if="unsplashImages.length" class="card-columns mt-3">
                            <div v-for="(image, $index) in unsplashImages" :key="$index" class="card border-0 bg-transparent">
                                <img
                                    :src="image.urls.small"
                                    :alt="image.alt_description"
                                    class="card-img bg-transparent"
                                    style="cursor: pointer"
                                    @click="selectUnsplashImage(image)"
                                />
                            </div>
                        </div>

                        <infinite-loading v-if="isSearchingUnsplash" :identifier="infiniteId" @infinite="fetchUnsplashImages" spinner="spiral">
                            <span slot="no-more"></span>
                            <div slot="no-results" class="mb-3">
                                No images found for "{{ searchKeyword }}"
                            </div>
                        </infinite-loading>
                    </div>

                    <div v-if="!isSearchingUnsplash && !unsplashImages.length">
                        <div v-if="selectedImageUrl && !selectedImagesForPond.length && !isReadyToAcceptUploads" class="selected-image">
                            <button type="button" class="close" data-dismiss="modal" aria-label="Close" @click.prevent="clearAndResetComponent">
                                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" class="icon-close-circle">
                                    <circle cx="12" cy="12" r="10" class="primary"/>
                                    <path class="fill-bg" d="M13.41 12l2.83 2.83a1 1 0 0 1-1.41 1.41L12 13.41l-2.83 2.83a1 1 0 1 1-1.41-1.41L10.59 12 7.76 9.17a1 1 0 0 1 1.41-1.41L12 10.59l2.83-2.83a1 1 0 0 1 1.41 1.41L13.41 12z"/>
                                </svg>
                            </button>
                            <img :src="selectedImageUrl" class="w-100 rounded mb-3"/>
                        </div>

                        <div class="col-12" :hidden="!selectedImagesForPond.length && !selectedImageUrl">
                            <div class="form-group row">
                                <label class="font-weight-bold text-uppercase text-muted small">Caption</label>
                                <input
                                    type="text"
                                    class="form-control border-0 px-0 bg-transparent"
                                    v-model="selectedImageCaption"
                                    :placeholder="trans.posts.forms.editor.images.picker.uploader.caption.placeholder"
                                    ref="caption"/>
                            </div>
                        </div>
                    </div>
                </div>
                <div v-if="!unsplashImages.length" class="modal-footer">
                    <button
                        class="btn btn-link btn-block text-muted font-weight-bold text-decoration-none"
                        @click="clickDone"
                        data-dismiss="modal">
                        {{ trans.buttons.general.done }}
                    </button>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    import _ from "lodash"
    import {mapState} from 'vuex'
    import Unsplash, {toJson} from 'unsplash-js'
    import InfiniteLoading from 'vue-infinite-loading'
    import vueFilePond from 'vue-filepond'

    import 'filepond/dist/filepond.min.css'
    import 'filepond-plugin-image-preview/dist/filepond-plugin-image-preview.min.css'

    import FilePondPluginImageValidateSize from 'filepond-plugin-image-validate-size'
    import FilePondPluginFileValidateType from 'filepond-plugin-file-validate-type'
    import FilePondPluginImagePreview from 'filepond-plugin-image-preview'

    const FilePond = vueFilePond(
        FilePondPluginFileValidateType,
        FilePondPluginImagePreview,
        FilePondPluginImageValidateSize,
    );

    export default {
        name: 'featured-image-modal',

        components: {
            InfiniteLoading,
            FilePond
        },

        data() {
            return {
                isReadyToAcceptUploads: true,
                searchKeyword: '',
                unsplashKey: Canvas.unsplash,
                unsplashPage: 1,
                unsplashPerPage: 12,
                unsplashImages: [],
                infiniteId: +new Date(),
                isSearchingUnsplash: false,
                selectedImageUrl: null,
                selectedImageCaption: '',
                selectedImagesForPond: [],
                galleryModalClasses: ['modal-xl', 'modal-dialog-scrollable'],
                path: Canvas.path,
                trans: JSON.parse(Canvas.lang),
            }
        },

        mounted() {
            this.selectedImageUrl = this.activePost.featured_image
            this.selectedImageCaption = this.activePost.featured_image_caption
            this.isReadyToAcceptUploads = _.isEmpty(this.activePost.featured_image);
        },

        watch: {
            searchKeyword: _.debounce(function (val) {
                if (val === '') {
                    this.isReadyToAcceptUploads = !this.selectedImageUrl;
                    this.isSearchingUnsplash = false
                    this.unsplashPage = 1
                    this.unsplashImages = []
                    this.infiniteId += 1
                    this.$refs.modal.classList.remove(...this.galleryModalClasses)
                } else {
                    this.isReadyToAcceptUploads = false
                    this.isSearchingUnsplash = true
                    this.unsplashPage = 1
                    this.unsplashImages = []
                    this.infiniteId += 1
                    this.$refs.modal.classList.add(...this.galleryModalClasses)
                }
            }, 1000),
        },

        methods: {
            fetchUnsplashImages($state) {
                const unsplash = new Unsplash({accessKey: this.unsplashKey})
                unsplash.search.photos(this.searchKeyword, this.unsplashPage, this.unsplashPerPage)
                    .then(toJson)
                    .then(json => {
                        if (!_.isEmpty(json.results)) {
                            this.unsplashImages.push(...json.results)
                            this.unsplashPage += 1;

                            $state.loaded()
                        } else {
                            $state.complete()
                        }
                    });
            },

            selectUnsplashImage(image) {
                const unsplash = new Unsplash({accessKey: this.unsplashKey});

                // We must trigger a download to properly attribute traffic to the source
                // https://help.unsplash.com/en/articles/2511258-guideline-triggering-a-download
                unsplash.photos.downloadPhoto(image);

                this.selectedUnsplashImage = image
                this.selectedImageUrl = image.urls.regular
                this.selectedImageCaption = this.buildImageCaption(image)
                this.unsplashImages = []
                this.unsplashPage = 1
                this.searchKeyword = ''
                this.$refs.modal.classList.remove(...this.galleryModalClasses)

                this.$emit('changed', {
                    url: image.urls.regular,
                    caption: this.buildImageCaption(image),
                })
            },

            buildImageCaption(image) {
                return this.trans.posts.forms.editor.images.picker.caption.by +
                    ' <a href="' +
                    image.user.links.html +
                    '" target="_blank">' +
                    image.user.name +
                    '</a> ' +
                    this.trans.posts.forms.editor.images.picker.caption.on +
                    ' <a href="https://unsplash.com" target="_blank">Unsplash</a>'
            },

            processedFromFilePond() {
                this.isReadyToAcceptUploads = true
                this.selectedImageUrl = document.getElementsByName('featuredImagePond')[0].value
            },

            removedFromFilePond() {
                this.isReadyToAcceptUploads = true
                this.selectedImagesForPond = []
                this.selectedImageUrl = null
            },

            clickDone() {
                this.activePost.featured_image = !_.isEmpty(this.selectedImageUrl) ? this.selectedImageUrl : ''
                this.activePost.featured_image_caption = !_.isEmpty(this.selectedImageCaption) ? this.selectedImageCaption : ''

                this.$parent.save()
            },

            clearAndResetComponent() {
                this.selectedImagesForPond = []
                this.selectedImageUrl = null
                this.selectedImageCaption = ''
                this.isReadyToAcceptUploads = true
                this.isSearchingUnsplash = false
                this.unsplashImages = []
                this.unsplashPage = 1
                this.searchKeyword = ''
                this.$refs.modal.classList.remove(...this.galleryModalClasses)
            },

            closeModal() {
                this.clearAndResetComponent()
                this.$refs.modal.hide
            }
        },

        computed: {
            ...mapState(['activePost']),

            getServerOptions() {
                return {
                    url: this.getUploadPath,
                    headers: {
                        'X-CSRF-TOKEN': document.head.querySelector('meta[name="csrf-token"]').content
                    }
                }
            },

            getUploadPath() {
                return '/' + this.path + '/api/media/uploads'
            },

            getPlaceholderLabel() {
                return '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="25" class="icon-cloud-upload"><path class="secondary" d="M18 14.97c0-.76-.3-1.51-.88-2.1l-3-3a3 3 0 0 0-4.24 0l-3 3A3 3 0 0 0 6 15a4 4 0 0 1-.99-7.88 5.5 5.5 0 0 1 10.86-.82A4.49 4.49 0 0 1 22 10.5a4.5 4.5 0 0 1-4 4.47z"/><path class="secondary" d="M11 14.41V21a1 1 0 0 0 2 0v-6.59l1.3 1.3a1 1 0 0 0 1.4-1.42l-3-3a1 1 0 0 0-1.4 0l-3 3a1 1 0 0 0 1.4 1.42l1.3-1.3z"/></svg> Drop files or click here to upload'
            },
        }
    }
</script>

<style lang="scss">
    @import '../../../sass/variables';

    .filepond--drop-label,
    .filepond--drop-label label {
        cursor: pointer;
    }

    .filepond--drop-label {
        color: $gray-700;
    }

    .filepond--panel-root {
        background-color: $gray-400;
    }

    .filepond--panel-root {
        border-radius: $border-radius;
    }

    .filepond--item-panel {
        border-radius: $border-radius;
    }

    [data-filepond-item-state*='error'] .filepond--item-panel,
    [data-filepond-item-state*='invalid'] .filepond--item-panel {
        background-color: $red;
    }

    [data-filepond-item-state='processing-complete'] .filepond--item-panel {
        background-color: $green;
    }

    .selected-image button {
        position: absolute;
        top: 25px;
        right: 27px;
    }
</style>