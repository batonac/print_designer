<template>
	<div
		:style="[postionalStyles(startX, startY, width, height), ]"
		:ref="setElements(object, index)"
		:class="[MainStore.getCurrentElementsId.includes(id) && 'active-elements']"
		@mousedown.left="handleMouseDown($event, object)"
		@dblclick.stop="handleDblClick($event, object)"
		@mouseup="handleMouseUp($event, object)"
	>
		<div
			v-if="barcodeSvg"
			:style="[
				widthHeightStyle(width, height),
				style,
			]"
			:class="['barcode', classes]"
			:key="id"
			v-html="barcodeSvg"
		></div>
		<div class="fallback-image" v-else>
			<div class="content">
				<span v-if="width >= 100 || height >= 100">Please Double click to select Barcode</span>
			</div>
		</div>
		<BaseResizeHandles
			v-if="
				MainStore.activeControl == 'mouse-pointer' &&
				MainStore.getCurrentElementsId.includes(id)
			"
		/>
	</div>
</template>

<script setup>
import { useMainStore } from "../../store/MainStore";
import { toRefs, ref, watch } from "vue";
import { useElement } from "../../composables/Element";
import {
	postionalStyles,
	setCurrentElement,
	lockAxis,
	cloneElement,
	deleteCurrentElements,
	widthHeightStyle,
} from "../../utils";
import { useDraw } from "../../composables/Draw";
import BaseResizeHandles from "./BaseResizeHandles.vue";

const MainStore = useMainStore();
const props = defineProps({
	object: {
		type: Object,
		required: true,
	},
	index: {
		type: Number,
		required: true,
	},
});
const { id, value, dynamicContent, barcodeFormat, barcodeColor, barcodeBackgroundColor, startX, startY, width, height, style, classes } = toRefs(
	props.object
);
const barcodeSvg = ref(null);
watch(()=> dynamicContent.value, () => {
	if (dynamicContent.value) {
		value.value = dynamicContent.value[0]?.value || "";
	}
}, { deep:true, immediate: true });

watch(() => [value.value, barcodeFormat.value, barcodeColor.value, barcodeBackgroundColor.value], async () => {
	if (!value.value || !barcodeFormat.value) return;
	try {
		const options = {
			background : barcodeBackgroundColor.value || "#ffffff",
			quiet_zone: 1,
		};
		if (barcodeFormat.value == "qrcode") {
			options["module_color"] = barcodeColor.value || "#000000";
		} else {
			options["foreground"] = barcodeColor.value || "#000000";
		}
		let barcode = await frappe.call(
			"print_designer.print_designer.page.print_designer.print_designer.get_barcode",
			{
				barcode_format: barcodeFormat.value,
				barcode_value: value.value,
				options,
			}
		);
		barcodeSvg.value = barcode.message.value;
	} catch (e) {
		barcodeSvg.value = null;
	}
}, { immediate: true });

const { setElements } = useElement({
	draggable: true,
	resizable: true,
});

const { drawEventHandler, parameters } = useDraw();

const handleMouseDown = (e, element = null) => {
	e.stopPropagation();
	if (MainStore.openModal) return;
	lockAxis(element, e.shiftKey);
	MainStore.isMoveStart = true;
	if (MainStore.activeControl == "mouse-pointer" && e.altKey) {
		element && setCurrentElement(e, element);
		cloneElement();
	} else {
		element && setCurrentElement(e, element);
	}
	MainStore.setActiveControl("MousePointer");
	MainStore.currentDrawListener = {
		drawEventHandler,
		parameters,
	};
	drawEventHandler.mousedown(e);
};

const handleMouseUp = (e, element = null) => {
	if (
		MainStore.lastCreatedElement &&
		!MainStore.openModal &&
		!MainStore.isMoved &&
		MainStore.isDrawing
	) {
		if (!MainStore.modalLocation.isDragged) {
			clientX = e.clientX;
			clientY = e.clientY;
			if (clientX - 250 > 0) clientX = clientX - 250;
			if (clientY - 180 > 0) clientY = clientY - 180;
			MainStore.modalLocation.x = clientX;
			MainStore.modalLocation.y = clientY;
		}
		MainStore.getCurrentElementsId.forEach((element) => {
			delete MainStore.currentElements[element];
		});
		MainStore.currentElements[MainStore.lastCreatedElement.id] = MainStore.lastCreatedElement;
		MainStore.openModal = true;
	} else if (
		MainStore.lastCloned &&
		!MainStore.isMoved &&
		MainStore.activeControl == "mouse-pointer"
	) {
		deleteCurrentElements();
	} else {
		MainStore.activeControl == "barcode" && (MainStore.openBarcodeModal = element);
	}
	MainStore.setActiveControl("MousePointer");
	MainStore.isMoved = MainStore.isMoveStart = false;
	MainStore.lastCloned = null;
};

const handleDblClick = (e, element) => {
	element && setCurrentElement(e, element);
	MainStore.openBarcodeModal = element;
};
</script>

<style lang="scss" scoped>
.fallback-image {
	width: 100%;
	user-select: none;
	height: 100%;
	display: flex;
	align-items: center;
	justify-content: center;
	background-color: var(--bg-color);
	.content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;

		span {
			font-size: smaller;
			text-align: center;
		}
	}
}
</style>
