<template>
	<div class="MalInputSize2d">
		<MalExpButton
			v-if="!isValueSeparated"
			:value="size"
			:compact="true"
			@select="$emit('select', $event)"
		/>
		<template v-if="isValueSeparated">
			<MalInputNumber
				class="MalInputSize2d__el"
				:value="nonReactiveValues[0]"
				:compact="true"
				@input="(...$event) => onInputElement(0, ...$event)"
				@select="$emit('select', $event)"
				@end-tweak="$emit('end-tweak')"
			/>
			<MalInputNumber
				class="MalInputSize2d__el"
				:value="nonReactiveValues[1]"
				:compact="true"
				@input="(...$event) => onInputElement(0, ...$event)"
				@select="$emit('select', $event)"
				@end-tweak="$emit('end-tweak')"
			/>
			<button
				class="MalInputSize2d__link-button fas"
				:class="{'fa-link': ratio !== false, 'fa-unlink': ratio === false}"
				@click="onClickLinkButton"
			/>
		</template>
		<template v-else>
			<InputNumber
				class="MalInputSize2d__el exp"
				:value="evaluated[0]"
				@input="onInputEvaluatedElement(0, $event)"
				@end-tweak="$emit('end-tweak')"
			/>
			<InputNumber
				class="MalInputSize2d__el exp"
				:value="evaluated[1]"
				@input="onInputEvaluatedElement(1, $event)"
				@end-tweak="$emit('end-tweak')"
			/>
		</template>
	</div>
</template>

<script lang="ts">
import {
	defineComponent,
	toRef,
	SetupContext,
	computed,
} from '@vue/composition-api'
import {
	MalSeq,
	isSeq,
	MalSymbol,
	isSymbol,
	isVector,
	getEvaluated,
	isList,
	isSymbolFor,
	MalVal,
	cloneExp,
	createList,
	symbolFor,
} from '@/mal/types'
import MalInputNumber from './MalInputNumber.vue'
import MalExpButton from './MalExpButton.vue'
import {InputNumber, InputTranslate} from '@/components/inputs'
import {useNumericVectorUpdator} from '@/components/use'
import {reverseEval} from '@/mal/utils'
import {NonReactive, nonReactive} from '@/utils'

interface Props {
	value: NonReactive<MalSeq | MalSymbol>
}

export default defineComponent({
	name: 'MalInputSize2d',
	components: {MalInputNumber, MalExpButton, InputNumber},
	props: {
		value: {
			required: true,
			validator: x =>
				x instanceof NonReactive && (isSeq(x.value) || isSymbol(x.value)),
		},
	},
	setup(props: Props, context: SetupContext) {
		const isSizeFunc = computed(
			() =>
				isList(props.value.value) &&
				isSymbolFor(props.value.value[0], 'vec2/size')
		)

		const size = computed(() => {
			const value = props.value.value
			if (isSizeFunc.value) {
				return nonReactive((value as MalSeq)[1])
			} else {
				return props.value
			}
		})

		const ratio = computed(() => {
			const value = props.value.value
			if (isSizeFunc) {
				return (value as MalSeq)[2] as number | false
			} else {
				return false
			}
		})

		const isValueSeparated = computed(() => isVector(size.value.value))

		const nonReactiveValues = computed(() => {
			if (!isValueSeparated.value) {
				return []
			} else {
				return Array.from(size.value.value as MalSeq).map(nonReactive)
			}
		})

		const evaluated = computed(() => getEvaluated(size.value.value) as number[])

		function onInputElement(
			index: number,
			v: NonReactive<MalVal>,
			num: number
		) {
			if (!isValueSeparated.value) {
				return
			}

			const newSize = cloneExp(size.value.value as MalSeq)
			const prevValue = newSize[index]
			newSize[index] = v.value

			const r = evaluated.value[1] / evaluated.value[0]

			if (ratio.value !== false) {
				const anotherIndex = index === 0 ? 1 : 0

				let anotherValue: MalVal = evaluated.value[anotherIndex]
				if (r === 0) {
					anotherValue = anotherIndex === 0 ? anotherValue : 0
				} else if (Math.abs(r) === Infinity) {
					anotherValue = anotherIndex === 0 ? 0 : anotherValue
				} else {
					anotherValue = anotherIndex === 0 ? (1 / r) * num : r * num
				}

				anotherValue = reverseEval(anotherValue, newSize[anotherIndex])
				newSize[anotherIndex] = anotherValue
			}

			const newExp = createList(
				symbolFor('vec2/size'),
				newSize,
				ratio.value === false ? false : r
			)
			context.emit('input', nonReactive(newExp))
		}

		function onInputEvaluatedElement(i: number, v: number) {
			const value = cloneExp(exp.value.value as MalSeq)
			value[i] = v
			const newExp = reverseEval(value, exp.value.value)
			context.emit('input', nonReactive(newExp))
		}

		function onClickLinkButton() {
			const newRatio =
				ratio.value === false ? evaluated.value[1] / evaluated.value[0] : false

			const newExp = createList(
				symbolFor('vec2/size'),
				size.value.value,
				newRatio
			)

			context.emit('input', nonReactive(newExp))
		}

		return {
			size,
			ratio,
			nonReactiveValues,
			isValueSeparated,
			evaluated,
			onInputElement,
			onInputEvaluatedElement,
			onClickLinkButton,
		}
	},
})
</script>

<style lang="stylus">
@import '../style/common.styl'

.MalInputSize2d
	display flex
	align-items center
	line-height $input-height

	&__el
		margin-right 0.6rem

	&__link-button
		padding 0
		width $button-height
		height @width !important
		color var(--button)
</style>