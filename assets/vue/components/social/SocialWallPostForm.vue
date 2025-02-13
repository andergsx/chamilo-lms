<template>
  <BaseCard plain>
    <form>
      <BaseInputTextWithVuelidate
        v-model="content"
        class="mb-2"
        :label="textPlaceholder"
        :vuelidate-property="v$.content"
      />

      <div class="mb-2">
        <BaseCheckbox
          v-if="allowCreatePromoted"
          id="is-promoted"
          v-model="isPromoted"
          :label="$t('Mark as promoted message')"
          name="is-promoted"
        />
      </div>

      <div class="flex mb-2">
        <BaseFileUpload
          v-model="attachment"
          :label="$t('File upload')"
          accept="image"
          size="small"
          @file-selected="v$.attachment.$touch()"
        />

        <BaseButton
          :label="$t('Post')"
          class="ml-auto"
          type="primary"
          icon="send"
          size="small"
          @click="sendPost"
        />
      </div>
    </form>
  </BaseCard>
</template>

<script setup>
import {inject, onMounted, reactive, ref, toRefs, watch} from "vue";
import {useStore} from "vuex";
import {SOCIAL_TYPE_PROMOTED_MESSAGE, SOCIAL_TYPE_WALL_POST} from "./constants";
import useVuelidate from "@vuelidate/core";
import {required} from "@vuelidate/validators";
import {useI18n} from "vue-i18n";
import BaseCard from "../basecomponents/BaseCard.vue";
import BaseButton from "../basecomponents/BaseButton.vue";
import BaseFileUpload from "../basecomponents/BaseFileUpload.vue";
import BaseCheckbox from "../basecomponents/BaseCheckbox.vue";
import BaseInputTextWithVuelidate from "../basecomponents/BaseInputTextWithVuelidate.vue";

const emit = defineEmits(['post-created']);
const store = useStore();
const {t} = useI18n();

const user = inject('social-user');

const currentUser = store.getters['security/getUser'];
const userIsAdmin = store.getters['security/isAdmin'];

const postState = reactive({
  content: '',
  attachment: null,
  isPromoted: false,
  textPlaceholder: '',
});

const {content, attachment, isPromoted, textPlaceholder} = toRefs(postState)

const v$ = useVuelidate({
  content: {required},
}, postState);

watch(() => user.value, () => {
  showTextPlaceholder();

  showCheckboxPromoted();
});

onMounted(() => {
  showTextPlaceholder();

  showCheckboxPromoted();
});

function showTextPlaceholder() {
  postState.textPlaceholder = currentUser['@id'] === user.value['@id']
    ? t('What are you thinking about?')
    : t('Write something to {0}', [user.value.fullName]);
}

const allowCreatePromoted = ref(false);

function showCheckboxPromoted() {
  allowCreatePromoted.value = userIsAdmin && currentUser['@id'] === user.value['@id'];
}

async function sendPost() {
  v$.value.$touch();

  if (!postState.content.trim()) {
    return;
  }

  if (v$.value.$error) {
    return;
  }

  try {
    await store.dispatch('socialpost/create', {
      content: postState.content,
      type: postState.isPromoted ? SOCIAL_TYPE_PROMOTED_MESSAGE : SOCIAL_TYPE_WALL_POST,
      sender: currentUser['@id'],
      userReceiver: currentUser['@id'] === user.value['@id'] ? null : user.value['@id'],
    });

    if (postState.attachment) {
      const formData = new FormData();
      formData.append('file', postState.attachment);
      const post = store.state.socialpost.created;
      await store.dispatch('messageattachment/createWithFormData', {
        postId: post.id,
        file: formData
      });
    }

    postState.content = '';
    postState.attachment = null;
    postState.isPromoted = false;
    v$.value.$reset();

    emit('post-created');
  } catch (error) {
    console.error("There was an error creating the post:", error);
  }
}
</script>
