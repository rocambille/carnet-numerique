<template>
  <Modal
    @close="$emit('close')"
    @submit="newPublication"
    :read_only="read_only"
    :typeOfModal="'EditMeta'"
    :askBeforeClosingModal="askBeforeClosingModal"
    >
    <template slot="header">
      <span>{{ $t(publidata.template) }}</span>
    </template>

    <template slot="sidebar">

<!-- Human name -->
      <div class="margin-bottom-small">
        <label>{{ $t('name') }}</label>
        <input type="text" v-model="publidata.name" required autofocus autoselect>
      </div>

<!-- Template -->
      <!-- <div class="margin-bottom-small">
        <label>{{ $t('format') }}</label>
        <select v-model="publidata.template">
          <option v-for="template in $root.state.list_of_publications_templates"
            :key="template"
            :value="template"
            v-html="$t(template)"
          />
        </select>
      </div> -->

<!-- Keywords -->
      <div class="margin-bottom-small">
        <label>{{ $t('keywords') }}</label>
        <TagsInput @tagsChanged="newTags => publidata.keywords = newTags"/>
      </div>

<!-- Author(s) -->
      <div class="margin-bottom-small">
        <label>{{ $t('author') }}</label><br>
        <AuthorsInput
          :currentAuthors="publidata.authors"
          @authorsChanged="newAuthors => publidata.authors = newAuthors"
        />
        <small>{{ $t('author_instructions') }}</small>
      </div>


    </template>

    <template slot="submit_button">
      {{ $t('create') }}
    </template>

  </Modal>
</template>
<script>
import Modal from './BaseModal.vue';
import TagsInput from '../subcomponents/TagsInput.vue';
import AuthorsInput from '../subcomponents/AuthorsInput.vue';

export default {
  props: {
    read_only: Boolean,
    default_name: {
      default: '',
      type: String
    },
    default_template: {
      default: 'page_by_page',
      type: String
    }
  },
  components: {
    Modal,
    TagsInput,
    AuthorsInput
  },
  data() {
    return {
      publidata: {
        name: this.default_name,
        template: this.default_template,
        keywords: [],
        authors: this.$root.settings.current_author.hasOwnProperty('name') ? [{ name: this.$root.settings.current_author.name }] : [],
      }
    };
  },
  watch: {
    'publidata': {
      handler: function() {
        this.askBeforeClosingModal = true;
      },
      deep: true
    }
  },
  mounted() {
  },
  computed: {},
  methods: {
    newPublication: function(event) {
      if (this.$root.state.dev_mode === 'debug') {
        console.log('METHODS • CreatePublication: newPublication');
      }
      let name = this.publidata.name;

      function getAllPubliNames() {
        let allPublicationsName = [];
        for (let slugPubliName in window.store.publications) {
          let publiName = window.store.publications[slugPubliName].name;
          allPublicationsName.push(publiName);
        }
        return allPublicationsName;
      }
      let allPublicationsName = getAllPubliNames();

      // check if project name (not slug) already exists
      if (allPublicationsName.indexOf(this.publidata.name) >= 0) {
        // invalidate if it does
        this.$alertify
          .closeLogOnClick(true)
          .delay(4000)
          .error(this.$t('notifications.publi_name_exists'));

        return false;
      }

      let publidata = {
        name,
        template: this.publidata.template,
        authors: this.publidata.authors,
        keywords: this.publidata.keywords
      }

      if(publidata.template === 'page_by_page') {
        publidata.pages = [{
          id: +new Date() + '_' + (Math.random().toString(36) + '00000000000000000').slice(2, 3)
        }];
        publidata.width = 210;
        publidata.height = 297;
      }

      this.$eventHub.$on('socketio.folder_created_or_updated', this.newPublicationCreated);
      this.$root.createFolder({ type: 'publications', data: publidata });      
    },
    newPublicationCreated: function(pdata) {
      if(pdata.id === this.$root.justCreatedFolderID) {
        this.$eventHub.$off('socketio.folder_created_or_updated', this.newPublicationCreated);
        this.$root.justCreatedFolderID = false;
        this.$nextTick(() => {
          this.$emit('close', '');
          this.$root.openPublication(pdata.slugFolderName);
        });
      }
    }
  }
};
</script>