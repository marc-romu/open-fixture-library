<template>
  <A11yDialog
    id="help-wanted-dialog"
    ref="dialog"
    :is-alert-dialog="state === `loading`"
    :shown="context !== undefined"
    :title="title"
    @hide="onHide()">

    <form v-if="state === `ready` && context !== undefined" action="#" @submit.prevent="onSubmit()">
      <LabeledValue
        v-if="location !== null"
        :value="location"
        label="Location" />

      <LabeledValue
        v-if="context.helpWanted !== null"
        label="Problem description">
        <span v-html="context.helpWanted" />
      </LabeledValue>

      <LabeledInput name="message" label="Message">
        <textarea v-model="message" name="message" />
      </LabeledInput>

      <LabeledInput
        name="github-username"
        label="GitHub username"
        hint="If you want to be mentioned in the issue.">
        <input v-model="githubUsername" type="text" name="github-username">
      </LabeledInput>

      <div class="button-bar right">
        <button :disabled="message === ``" type="submit" class="primary">Send information</button>
      </div>
    </form>

    <template v-else-if="state === `loading`">
      Uploading…
    </template>

    <template v-else-if="state === `success`">
      Your information was successfully uploaded to GitHub (see the <a :href="issueUrl" target="_blank" rel="noopener">issue</a>). The fixture will be updated as soon as your information has been reviewed. Thank you for your contribution!

      <div class="button-bar right">
        <a href="#" class="button secondary" @click.prevent="hide()">Close</a>
        <a :href="issueUrl" class="button primary" target="_blank">See issue</a>
      </div>
    </template>

    <template v-else-if="state === `error`">
      <span>Unfortunately, there was an error while uploading. Please copy the following data and manually submit it.</span>

      <textarea :value="errorData" readonly />

      <div class="button-bar right">
        <a href="#" class="button secondary" @click.prevent="hide()">Close</a>
        <a :href="mailtoUrl" class="button secondary" target="_blank">Send email</a>
        <a
          href="https://github.com/OpenLightingProject/open-fixture-library/issues/new"
          class="button primary"
          target="_blank"
          rel="noopener">
          Create issue on GitHub
        </a>
      </div>
    </template>

  </A11yDialog>
</template>

<script>
import { objectProp, stringProp } from 'vue-ts-types';
import A11yDialog from './A11yDialog.vue';
import LabeledInput from './LabeledInput.vue';
import LabeledValue from './LabeledValue.vue';

export default {
  components: {
    A11yDialog,
    LabeledInput,
    LabeledValue,
  },
  model: {
    prop: `context`,
  },
  props: {
    type: stringProp().required,
    context: objectProp().optional,
  },
  data: () => {
    return {
      state: `ready`,
      message: ``,
      githubUsername: ``,
      issueUrl: null,
      error: null,
    };
  },
  computed: {
    title() {
      if (this.state === `loading`) {
        return `Sending your message…`;
      }

      if (this.state === `success`) {
        return `Message sent`;
      }

      if (this.state === `error`) {
        return `Failed to send message`;
      }

      if (this.type === `plugin`) {
        return `Improve plugin`;
      }

      return `Improve fixture`;
    },
    location() {
      if (this.type === `capability`) {
        const capability = this.context;
        const channel = capability._channel;
        return `Channel "${channel.key}" → Capability "${capability.name}" (${capability.rawDmxRange})`;
      }

      return null;
    },
    fixture() {
      if (this.type === `fixture`) {
        return this.context;
      }

      if (this.type === `capability`) {
        return this.context._channel.fixture;
      }

      return null;
    },
    sendObject() {
      const sendObject = {
        type: this.type,
        location: this.location,
        helpWanted: this.context.helpWanted,
        message: this.message,
        githubUsername: this.githubUsername === `` ? null : this.githubUsername,
      };

      if (this.type === `plugin`) {
        sendObject.context = this.context.key;
      }
      else {
        const manufacturerKey = this.fixture.manufacturer.key;
        const fixtureKey = this.fixture.key;

        sendObject.context = `${manufacturerKey}/${fixtureKey}`;
      }

      return sendObject;
    },
    errorData() {
      return `${JSON.stringify(this.sendObject, null, 2)}\n\n${this.error}`;
    },
    mailtoUrl() {
      const subject = `Feedback for ${this.sendObject.type} '${this.sendObject.context}'`;

      const mailBodyData = {
        'Problem location': this.location,
        'Problem description': this.context.helpWanted,
        'Message': this.message,
      };

      const body = Object.entries(mailBodyData).filter(
        ([key, value]) => value !== null,
      ).map(([key, value]) => {
        const separator = value.includes(`\n`) ? `\n` : ` `;
        return `${key}:${separator}${value}`;
      }).join(`\n`);

      return `mailto:flo@open-fixture-library.org?subject=${encodeURIComponent(subject)}&body=${encodeURIComponent(body)}`;
    },
  },
  mounted() {
    if (localStorage) {
      this.githubUsername = localStorage.getItem(`prefillGithubUsername`) || ``;
    }
  },
  methods: {
    async onSubmit() {
      this.state = `loading`;
      localStorage.setItem(`prefillGithubUsername`, this.githubUsername);

      try {
        const response = await this.$axios.post(`/api/v1/submit-feedback`, this.sendObject);

        if (response.data.error) {
          throw new Error(response.data.error);
        }

        this.issueUrl = response.data.issueUrl;
        this.state = `success`;
      }
      catch (error) {
        console.error(`There was a problem with the request.`, error);
        this.error = error.message;
        this.state = `error`;
      }
    },
    hide() {
      this.$refs.dialog.$emit(`hide`);
    },
    onHide() {
      if (this.state === `success`) {
        this.message = ``;
      }

      this.state = `ready`;
      this.issueUrl = null;
      this.error = null;
      this.$emit(`input`, undefined);
    },
  },
};
</script>
