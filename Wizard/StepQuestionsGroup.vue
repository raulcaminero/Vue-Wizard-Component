<template>
  <div>
    <div class="container"
         v-if="stepType=='message'">
      <div class="row justify-content-center align-self-center">
        <div class="featureIcon">
          <span><i :class="featureIcon"></i></span>
        </div>
      </div>
      <div class="row justify-content-center align-self-center">
        <div class="confirmationTitle col-8"
             v-html="confirmationTitle"
             v-if="confirmationTitle"></div>
      </div>
      <div class="row justify-content-center align-self-center">
        <div class="confirmationMessage"
             v-html="confirmationMessage"
             v-if="confirmationMessage"></div>
      </div>
    </div>
    <div v-else>
      <div v-for="question in questions"
           :key="question.number">
        <question :number="question.number" 
                  :sub-number="question.subNumber"
                  :description="question.description" 
                  :answer-control-type="question.answerControlType"
                  :error-message="question.errorMessage"
                  :warning-message="question.warningMessage"
                  :on-change="question.onChange"
                  :get-default-value="question.getDefaultValue"
                  :get-options="question.getOptions"
                  :is-required="question.isRequired"
                  :require-error-message="question.requireErrorMessage"
                  :ref="'question'+question.number"
                  :visible="question.isVisible"
                  :display-number="question.displayNumber"                 
                  @questionChange="onQuestionChange"
                  :picker-min-date="question.pickerMinDate"
                  :fields="question.fields"
                  :field-caption="question.fieldCaption"   
                  :notes="question.notes"/>     
      </div>
    </div>
  </div>
</template>

<script>
import Question from "./WizardQuestion";
export default {
	components: {
		Question
	},
	props: {
		questions: Array,
		featureIcon: String,
		confirmationTitle: String,
		confirmationMessage: String,
		stepType: String
	},
	data() {
		return {
			isValid: true
		};
	},
	watch: {
		questions: {
			handler: function(newQuestions) {
				let hasInvalidQuestion = newQuestions.some(question => {
					let questionElement = this.$refs[
						"question" + question.number
					][0];
					return (
						!questionElement.validationResult.isValid ||
						questionElement.invalidSelection
					);
				});
				this.isValid = !hasInvalidQuestion;
			},
			deep: true
		},
		isValid: {
			handler: function() {
				this.$emit("statusChange", this);
			},
			deep: true
		}
	},
	methods: {
		validate(onlyInvalidQuestion) {
			let validationResult = true;
			if (this.questions) {
				if (onlyInvalidQuestion) {
					this.questions
						.filter(question => question.hasAnError == true)
						.forEach(question => {
							let questionElement = this.$refs[
								"question" + question.number
							][0];
							let result = questionElement.validate();

							if (!result) {
								validationResult = false;
							}
						});
				} else {
					this.questions.forEach(question => {
						let questionElement = this.$refs[
							"question" + question.number
						][0];
						let result = questionElement.validate();

						if (!result) {
							validationResult = false;
						}
					});
				}
			}
			this.isValid = validationResult;

			return validationResult;
		},
		continue() {
			this.$emit("onContinueClick", this);
		},
		onQuestionChange() {
			this.validate(true);
			this.$emit("statusChange", this);
		},
		getQuestion(number) {
			let questionElement = this.$refs["question" + number];
			if (questionElement != null) {
				return questionElement[0];
			}
			return null;
		}
	}
};
</script>
<style scoped>
div.confirmationTitle {
	text-align: center;
	font: Bold 28px/30px Lato;
	letter-spacing: 0;
	color: #2db782;
	opacity: 1;
	margin-bottom: 10px;
}

div.confirmationMessage {
	text-align: center;
	font: Bold 28px/30px Lato;
	font-weight: 600;
	letter-spacing: 0;
	color: #11291b;
	opacity: 1;
}

.featureIcon {
	text-align: center;
	color: #2db782;
	font-size: 130px;
	border-radius: 50%;
}
.container {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	height: calc(100vh - 400px);
}
</style>
