<template>
    <div>
        <breadcrumb
            v-if="breadcrumbs"
            :url="breadcrumbs[0].url"
            :title="breadcrumbs[0].text"
        />

        <div class="flex items-center mb-3">
            <h1 class="flex-1">
                <div class="flex items-center">
                    <span v-html="title" />
                </div>
            </h1>

            <div v-if="!readOnly" class="hidden md:flex items-center">
                <save-button-options
                    v-if="!readOnly"
                    :preferences-prefix="preferencesPrefix"
                >
                    <button
                        class="btn-primary"
                        :disabled="isSaving"
                        @click.prevent="save"
                    >
                        {{ __('Save') }}
                    </button>
                </save-button-options>
            </div>
        </div>

        <publish-container
            ref="container"
            :name="publishContainer"
            :blueprint="blueprint"
            :values="values"
            :meta="meta"
            :errors="errors"
            @updated="values = $event"
        >
            <div>
                <component
                    v-for="component in components"
                    :key="component.id"
                    :is="component.name"
                    :container="container"
                    v-bind="component.props"
                    v-on="component.events"
                />

                <publish-tabs
                    :read-only="readOnly"
                    :enable-sidebar="shouldShowSidebar"
                    @updated="setFieldValue"
                    @meta-updated="setFieldMeta"
                    @focus="$refs.container.$emit('focus', $event)"
                    @blur="$refs.container.$emit('blur', $event)"
                >
                    <template #actions="{ shouldShowSidebar }">
                        <div class="card p-0" :class="{ 'mb-5': resourceHasRoutes && permalink }">
                            <div
                                v-if="resourceHasRoutes && permalink"
                                :class="{ hi: !shouldShowSidebar }"
                            >
                                <div class="p-2 flex items-center -mx-1">
                                    <a
                                        class="flex items-center justify-center btn-flat w-full mx-1 px-1"
                                        v-if="permalink"
                                        :href="permalink"
                                        target="_blank"
                                    >
                                        <svg-icon
                                            name="external-link"
                                            class="w-5 h-5 mr-1"
                                        />
                                        <span>{{ __('Visit URL') }}</span>
                                    </a>
                                </div>
                            </div>
                        </div>
                    </template>
                </publish-tabs>
            </div>
        </publish-container>

        <div class="md:hidden mt-3 flex items-center">
            <button
                v-if="!readOnly"
                class="btn-lg btn-primary w-full"
                :disabled="isSaving"
                @click.prevent="save"
            >
                {{ __('Save') }}
            </button>
        </div>
    </div>
</template>

<script>
import SaveButtonOptions from '../statamic/SaveButtonOptions.vue'
import HasPreferences from '../statamic/HasPreferences.js'

export default {
    components: {
        SaveButtonOptions,
    },

    mixins: [HasPreferences],

    props: {
        breadcrumbs: Array,
        initialBlueprint: Object,
        initialValues: Object,
        initialMeta: Object,
        initialTitle: String,
        action: String,
        method: String,
        resourceHasRoutes: Boolean,
        permalink: String,
        isCreating: {
            type: Boolean,
            default: false,
        },
        isInline: {
            type: Boolean,
            default: false,
        },
        publishContainer: String,
        readOnly: Boolean,
        resource: {
            type: Object,
            required: true,
        },
        createAnotherUrl: String,
        listingUrl: String,
    },

    data() {
        return {
            blueprint: this.initialBlueprint,
            values: this.initialValues,
            meta: this.initialMeta,
            title: this.initialTitle,
            preferencesPrefix: `runway.${this.resource.handle}`,

            errors: {},
            saving: false,
            containerWidth: null,
            saveKeyBinding: null,
            quickSave: false,
        }
    },

    computed: {
        enableSidebar() {
            return this.blueprint.tabs
                .map((section) => section.handle)
                .includes('sidebar')
        },

        shouldShowSidebar() {
            return this.enableSidebar
        },

        afterSaveOption() {
            return this.getPreference('after_save')
        },
    },

    mounted() {
        this.saveKeyBinding = this.$keys.bindGlobal(
            ['mod+s', 'mod+return'],
            (e) => {
                e.preventDefault()
                this.quickSave = true
                this.save()
            }
        )
    },

    created() {
        // If we're creating a resource through the 'Create' on a HasMany field somewhere, fill any fields...
        if (this.publishContainer.includes('relate-fieldtype-inline')) {
            this.values['from_inline_publish_form'] = true

            this.initialBlueprint.tabs.forEach((section) => {
                section.fields.forEach((field) => {
                    if (
                        field.type === 'belongs_to' &&
                        field.resource === window.Runway.currentResource
                    ) {
                        let alreadyExists = this.values[field.handle].includes(
                            window.Runway.currentRecord.id
                        )

                        if (!alreadyExists) {
                            this.values[field.handle].push(
                                window.Runway.currentRecord.id
                            )
                            this.meta[field.handle].data = [
                                window.Runway.currentRecord,
                            ]
                        }
                    }
                })
            })
        }
    },

    methods: {
        save() {
            this.saving = true
            this.clearErrors()

            this.$axios({
                method: this.method,
                url: this.action,
                data: this.values,
            })
                .then((response) => {
                    this.saving = false
                    this.$refs.container.saved()
                    this.$emit('saved', response)

                    let nextAction = this.quickSave
                        ? 'continue_editing'
                        : this.afterSaveOption

                    // If the user has opted to create another entry, redirect them to create page.
                    if (!this.isInline && nextAction === 'create_another') {
                        this.$nextTick(() => {
                            window.location = this.createAnotherUrl
                        })

                        return
                    }

                    // If the user has opted to go to listing (default/null option), redirect them there.
                    if (!this.isInline && nextAction === null) {
                        this.$nextTick(() => {
                            window.location = this.listingUrl
                        })

                        return
                    }

                    // Otherwise, leave them on the edit form (or redirect them to the edit form if they're creating a new model).
                    if (this.isCreating && this.publishContainer === 'base') {
                        this.$nextTick(() => {
                            window.location.href = response.data.redirect
                        })
                    } else {
                        this.quickSave = false

                        this.$toast.success(__('Saved'))
                    }
                })
                .catch((error) => this.handleAxiosError(error))
        },

        clearErrors() {
            this.error = null
            this.errors = {}
        },

        handleAxiosError(e) {
            this.saving = false

            if (e.response && e.response.status === 422) {
                const { message, errors } = e.response.data
                this.error = message
                this.errors = errors
                this.$toast.error(message)
            } else if (e.response) {
                this.$toast.error(e.response.data.message)
            } else {
                this.$toast.error(e || 'Something went wrong')
            }
        },

        setFieldValue(handle, value) {
            this.$refs.container.setFieldValue(handle, value)
        },

        setFieldMeta(handle, value) {
            this.$store.dispatch(
                `publish/${this.publishContainer}/setFieldMeta`,
                {
                    handle,
                    value,
                    user: Statamic.user.id,
                }
            )
        },
    },

    watch: {
        saving(saving) {
            this.$progress.loading(`runway-publish-form`, saving)
        },
    },

    destroyed() {
        this.saveKeyBinding.destroy()
    },
}
</script>
