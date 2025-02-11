<script>
	import {getContext, onMount} from 'svelte';
	import {is_function} from 'svelte/internal';
	import {createEventContent, toEventWithLocalDates, toViewWithLocalDates, setContent} from '@event-calendar/common';

	export let date;
	export let chunk;

	let {displayEventEnd, eventBackgroundColor, eventColor, eventContent, eventClick, eventDidMount,
		eventMouseEnter, eventMouseLeave, slotDuration, slotHeight, theme, _view, _intlEventTime,
		_interaction, _classes, _draggable} = getContext('state');

	let {_slotTimeLimits, _viewResources} = getContext('view-state');

	let el;
	let event;
	let display;
	let classes;
	let style;
	let content;
	let timeText;
	let dragged = false;

	$: event = chunk.event;

	$: {
		display = event.display;

		// Style
		let step = $slotDuration.seconds / 60;
		let offset = $_slotTimeLimits.min.seconds / 60;
		let start = (chunk.start - date) / 1000 / 60;
		let end = (chunk.end - date) / 1000 / 60;
		let top = (start - offset) / step * $slotHeight;
		let height = (end - start) / step * $slotHeight;
		let maxHeight = ($_slotTimeLimits.max.seconds / 60 - start) / step * $slotHeight;
		let bgColor = event.backgroundColor || $eventBackgroundColor || $eventColor;
		style =
			`top:${top}px;` +
			`min-height:${height}px;` +
			`height:${height}px;` +
			`max-height:${maxHeight}px;`
		;
		if (bgColor) {
			style += `background-color:${bgColor};`;
		}
		if (display === 'auto' || display === 'ghost') {
			style +=
				`z-index:${chunk.column + 1};` +
				`left:${100 / chunk.group.columns.length * chunk.column}%;` +
				`width:${100 / chunk.group.columns.length * 0.5 * (1 + chunk.group.columns.length - chunk.column)}%;`
			;
		}

		// Class
		classes = $_classes(display === 'background' ? $theme.bgEvent : $theme.event, event);
	}

	// Content
	$: [timeText, content] = createEventContent(chunk, $displayEventEnd, $eventContent, $theme, $_intlEventTime, $_view);

	onMount(() => {
		if (is_function($eventDidMount)) {
			$eventDidMount({
				event: toEventWithLocalDates(event),
				timeText,
				el,
				view: toViewWithLocalDates($_view)
			});
		}
	});

	function createHandler(fn, display) {
		return display !== 'preview' && is_function(fn)
			? jsEvent => fn({event: toEventWithLocalDates(event), el, jsEvent, view: toViewWithLocalDates($_view)})
			: undefined;
	}

	function createPointerDownHandler(draggable, display, event) {
		return display === 'auto' && draggable(event)
			? jsEvent => $_interaction.drag.startTimeGrid(event, el, jsEvent, _viewResources)
			: undefined;
	}
</script>

<div
	bind:this={el}
	class="{classes}"
	{style}
	use:setContent={content}
	on:click={createHandler($eventClick, display)}
	on:mouseenter={createHandler($eventMouseEnter, display)}
	on:mouseleave={createHandler($eventMouseLeave, display)}
	on:pointerdown={createPointerDownHandler($_draggable, display, event)}
></div>