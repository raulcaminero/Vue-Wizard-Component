<template>
  <li v-if="showStep"
      class="col step"
      :class="{ active: this.active, completed: this.completed }">
    <span>
      <span class="step-numbers"
            :class="{ active: this.active, completed: this.completed }">
        <span v-if="this.completed && !this.active"><i class="fas fa-check"></i></span>
        <span v-if="!this.completed || this.active">{{ numberDesc }}</span>
      </span>
      <span class="step-label mt-2"
            :class="{ active: this.active, completed: this.completed }">{{ stepDesc }}</span>
    </span>
  </li>
</template>

<script>
export default {
	props: {
		stepNumber: Number,
		numberDesc: String,
		stepDesc: String,
		active: Boolean,
		completed: Boolean,
		showStep: Boolean
	}
};
</script>

<style scoped>
.step-numbers {
	line-height: 36px !important;
	width: 38px;
	height: 38px;
	display: inline-block;
	background: #f2f2f2;
	border-radius: 50%;
	font: Bold 20px/30px Lato;
	letter-spacing: 0;
	color: #c9c9c9;
	text-decoration: none;
}
.step-numbers:hover {
	text-decoration: none;
}
.step-numbers.active {
	font: 900 22px/31px Lato;
	letter-spacing: 0;
	color: #ffffff;
	background: #2db782;
}
.step-numbers.completed {
	font: 900 22px/31px Lato;
	letter-spacing: 0;
	color: #ffffff;
	background: #2db782;
}

.step-label.completed {
	color: #2db782;
}

.step-label {
	display: block;
	font: 400 18px/22px Lato;
	letter-spacing: 0;
	color: #c9c9c9;
}

.bg-green .step-label {
	color: white;
}

.step-label.active {
	color: #2db782;
	font-weight: bold;
}

.step-label.completed {
	color: #2db782;
	font-weight: bold;
}

.step-pills > li > span {
	position: relative;
	display: block;
	padding: 0px 15px;
	text-align: center;
	text-decoration: none;
}

.step {
	padding-top: 10px;
	padding-bottom: 10px;
}
.step.completed,
.step.active {
	border-bottom: 3px solid #2db782;
}
</style>
