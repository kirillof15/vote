<template>
  <v-card class="mt-2 outlined vote-wrap" elevation="0" :loading="loading">
    <div class="border__block">
      <legend>Голосование</legend>
      <br />
      <card-radiogroup
        ref="radiogroup"
        v-model="voteField"
        :disabled="isVoteDisabled"
        required
        :column="true"
      >
        <card-radiobutton
          label="За"
          value="1"
          :disabled="isVoteDisabled"
          class="mb-2"
        ></card-radiobutton>
        <card-radiobutton
          label="Против"
          value="2"
          :disabled="isVoteDisabled"
          class="mb-2"
        ></card-radiobutton>
        <card-radiobutton
          label="Воздержался"
          value="3"
          :disabled="isVoteDisabled"
        ></card-radiobutton>
      </card-radiogroup>
      <br />
      <p class="font-weight-bold black--text">Комментарий</p>
      <card-textarea
        v-model="commentField"
        label=""
        :mobile="isMobile"
        :input-size="9"
        maxlength="4000"
        :readonly="isVoteDisabled"
        :clear-icon="!isVoteDisabled"
      ></card-textarea>
      <div class="button">
        <v-btn
          class="ma-2"
          outlined
          color="primary"
          @click="handleVote"
          :disabled="isVoteDisabled"
        >
          Проголосовать
        </v-btn>
      </div>
    </div>
  </v-card>
</template>

<script>
/* eslint-disable @typescript-eslint/typedef */
const ABSTAINED = "3";
const AGAINST = "2";
const GET_QUESTION = "GET";
const ALL_QUESTIONS = "ALL";
const SAVE_QUESTION = "SAVE";
const TIME_RECEIVE_QUESTION = 15000;
const INDEX = 1;

export default {
  mixins: ["card.mixin"],
  inject: ["getCurrentUserFromCard"],
  props: {
    card: { type: Object, default: null },
    lastSelectedRow: { type: Object, default: null },
    isDisabledVotesBtn: { type: Boolean, default: true },
    setVotesState: { type: Function, default: null },
    isStopVotes: { type: Boolean, default: false },
  },
  data() {
    return {
      isVoteDisabled: true,
      commentField: "",
      voteField: null,
      questionId: null,
      hasActiveMessagesFlag: false,
      loading: false,
    };
  },
  async created() {
    this.currentUser = this.getCurrentUserFromCard();
    this.hasActiveMessagesFlag = await this.hasActiveMessages(
      this.currentUser.id
    );
    this.isVoteDisabled = !this.hasActiveMessagesFlag;

    if (this.card.listAttr.GRK_AGENDA_QUESTIONS.length) {
      const vote = await this.getNewRowData(
        this.card.listAttr.GRK_AGENDA_QUESTIONS[0].ID,
        this.currentUser.id,
        null,
        null,
        GET_QUESTION
      );
      if (vote && vote.length) {
        this.voteField = String(vote[4].item);
        this.commentField = vote[5].item ? vote[5].item : "";
      }
    }
  },

  computed: {
    getRadiogroupElements() {
      return [...this.$refs.radiogroup.$el.getElementsByTagName("input")];
    },
  },

  watch: {
    async lastSelectedRow(value) {
      if (value) {
        this.questionId = this.card.listAttr.GRK_AGENDA_QUESTIONS.filter(
          (item) => item.QuestionNum === value.QuestionNum
        )[0].ID;
        const vote = await this.getNewRowData(
          this.questionId,
          this.currentUser.id,
          null,
          null,
          GET_QUESTION
        );

        if (vote && vote.length) {
          this.voteField = String(vote[4].item);
          this.commentField = vote[5].item ? vote[5].item : "";
          this.getRadiogroupElements[vote[4].item - INDEX].checked = true;
        } else {
          this.commentField = "";
          this.getRadiogroupElements.forEach(
            (input) => (input.checked = false)
          );
        }
      }
    },
    isDisabledVotesBtn(value) {
      this.isVoteDisabled = !value;
      this.commentField = "";
      this.getRadiogroupElements.forEach((input) => (input.checked = false));
    },
    disabledVotesStop(value) {
      this.isVoteDisabled = value;
    },
    isStopVotes(value) {
      this.isVoteDisabled = value;
    },
  },
  methods: {
    async handleVote() {
      if (!this.voteField) {
        this.$toasted.show(
          "Необходимо заполнить обязательные поля для голосования"
        );
        return;
      }
      if (this.voteField === AGAINST && this.commentField === "") {
        this.$toasted.show('Необходимо заполнить поле "Комментарий"');
        return;
      }
      if (!this.questionId) {
        this.$toasted.show("Необходимо выбрать вопрос");
        return;
      }
      this.isVoteDisabled = true;
      const voted = await this.vote(
        this.questionId,
        this.currentUser.id,
        this.voteField,
        this.commentField,
        SAVE_QUESTION
      );
      if (voted) {
        if (this.voteField === ABSTAINED) {
          this.$toasted.success('Результат изменен на "Воздержался"');
        } else {
          this.$toasted.success("Спасибо! Ваш голос учтен.");
        }
        const allVoted = await this.isAllQuestionsVoted(
          this.questionId,
          this.currentUser.id,
          this.voteField,
          this.commentField,
          ALL_QUESTIONS
        );
        const hasActiveMessagesFlag = await this.hasActiveMessages(
          this.currentUser.id
        );
        this.$emit("set-enabled-send-votes-result", {
          enabled: allVoted && hasActiveMessagesFlag,
        });
        await this.addUserVotesResult(this.card.id, this.currentUser.id);
        await this.addComment(this.card.id);
        const row = await this.getNewRowData(
          this.questionId,
          this.currentUser.id,
          this.voteField,
          this.commentField,
          GET_QUESTION
        );
        if (row) {
          this.$emit("add-row-after-voting", { row });
        }
      }
      this.isVoteDisabled = false;
    },

    async vote(questionId, userId, voteFor, comment, operation) {
      try {
        await this.execCommand(
          this.commandId.CallMethodCommand,
          this.getVoteParams(questionId, userId, voteFor, comment, operation)
        );
      } catch (e) {
        this.$toasted.error("Произошла ошибка при голосовании");
        return false;
      }
      return true;
    },

    async isAllQuestionsVoted(questionId, userId, voteFor, comment, operation) {
      const data = await this.execCommand(
        this.commandId.CallMethodCommand,
        this.getVoteParams(questionId, userId, voteFor, comment, operation)
      );
      return data?.item?.records[0]?.values[0]?.item === "+";
    },

    async addComment(docId) {
      await this.execCommand(this.commandId.CallMethodCommand, {
        entity: "DOCUMENT",
        method: "GRK_SP_GET_AGENDA_Q_COMMENT_BY_DOCID",
        fromRemote: true,
        parameters: [
          {
            type: "Long",
            name: "pDocID",
            value: docId,
          },
        ],
      });
    },

    async getNewRowData(questionId, userId, voteFor, comment, operation) {
      try {
        const data = await this.execCommand(
          this.commandId.CallMethodCommand,
          this.getVoteParams(questionId, userId, voteFor, comment, operation)
        );
        return data?.item?.records[0]?.values;
      } catch (e) {
        this.$toasted.error("Произошла ошибка при получении данных");
        return null;
      }
    },
  },
};
</script>

<style scoped>
.border__block {
  border: 1px solid #cccccc;
  padding: 10px;
}

legend {
  font-weight: bold;
  font-size: larger;
}

.button {
  display: flex;
  justify-content: flex-end;
}

.vote-wrap >>> {
  border-radius: 0;
}
</style>
