<script>
	import {getContext, onMount, afterUpdate, tick, createEventDispatcher} from 'svelte';
	import {is_function} from 'svelte/internal';
	import {
		ancestor,
		createEventContent,
		height,
		toEventWithLocalDates,
		toViewWithLocalDates,
		setContent,
		cloneDate
	} from '@event-calendar/common';

	export let chunk;
	export let longChunks = {};
	export let inPopup = false;

	let {dayMaxEvents, displayEventEnd, eventBackgroundColor, eventClick, eventColor, eventContent, eventDidMount,
		eventMouseEnter, eventMouseLeave, theme, _view, _intlEventTime, _interaction, _classes, _draggable} = getContext('state');
	let {_hiddenEvents} = getContext('view-state');

	const dispatch = createEventDispatcher();

	let el;
	let event;
	let classes;
	let style;
	let content;
	let timeText;
	let margin = 1;
	let hidden = false;
	let display;

	$: event = chunk.event;

	$: {
		display = event.display;

		// Class & Style
		let bgColor = event.backgroundColor || $eventBackgroundColor || $eventColor;
		style =
			`width:calc(${chunk.days * 100}% + ${(chunk.days - 1) * 7}px);` +
			`margin-top:${margin}px;`
		;
		if (bgColor) {
			style += `background-color:${bgColor};`;
		}
		if (hidden) {
			style += 'visibility:hidden;';
		}

		classes = $_classes($theme.event, event);
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

	afterUpdate(reposition);

	function createHandler(fn, display) {
		return display !== 'preview' && is_function(fn)
			? jsEvent => fn({event: toEventWithLocalDates(event), el, jsEvent, view: toViewWithLocalDates($_view)})
			: undefined;
	}

	function createClickHandler(fn, display) {
		let handler = createHandler(fn, display);
		return handler ? jsEvent => !jsEvent.ecClosingPopup && handler(jsEvent) : handler;
	}

	function createPointerDownHandler(draggable, display, event) {
		return display === 'auto' && draggable(event)
			? jsEvent => $_interaction.drag.startDayGrid(event, el, jsEvent, inPopup)
			: undefined;
	}

	function reposition() {
		if (!el || display === 'preview' || inPopup) {
			return;
		}
		chunk.top = 0;
		if (chunk.prev) {
			if (chunk.prev.bottom === undefined) {
				// 'prev' is not ready yet, try again later
				tick().then(reposition);
				return;
			}
			chunk.top = chunk.prev.bottom + 1;
		}
		chunk.bottom = chunk.top + height(el);
		let m = 1;
		let key = chunk.date.getTime();
		if (longChunks[key]) {
			for (let longChunk of longChunks[key]) {
				if (longChunk.bottom === undefined) {
					// 'longChunk' is not ready yet, try again later
					tick().then(reposition);
					return;
				}
				if (chunk.top < longChunk.bottom && chunk.bottom > longChunk.top) {
					let offset = longChunk.bottom - chunk.top + 1;
					m += offset;
					chunk.top += offset;
					chunk.bottom += offset;
				}
			}
		}
		margin = m;
		if ($dayMaxEvents === true) {
			hide();
		}
	}

	function hide() {
		if (!el) {
			return;
		}
		let dayEl = ancestor(el, 2);
		let h = height(dayEl) - height(dayEl.firstElementChild) - footHeight(dayEl);
		hidden = chunk.bottom > h;
		let date = cloneDate(chunk.date);
		let update = false;
		// Hide or show the event throughout all days
		for (let date of chunk.dates) {
			let hiddenEvents = $_hiddenEvents[date.getTime()];
			if (hiddenEvents) {
				let size = hiddenEvents.size;
				if (hidden) {
					hiddenEvents.add(chunk.event);
				} else {
					hiddenEvents.delete(chunk.event);
				}
				if (size !== hiddenEvents.size) {
					update = true;
				}
			}
		}
		if (update) {
			$_hiddenEvents = $_hiddenEvents;
		}
	}

	function footHeight(dayEl) {
		let h = 0;
		for (let i = 0; i < chunk.days; ++i) {
			h = Math.max(h, height(dayEl.lastElementChild));
			dayEl = dayEl.nextElementSibling;
			if (!dayEl) {
				break;
			}
		}
		return h;
	}

	$: if ($_hiddenEvents) {
		tick().then(reposition);
	}
</script>

<div
	bind:this={el}
	class="{classes}"
	{style}
	use:setContent={content}
	on:click={createClickHandler($eventClick, display)}
	on:mouseenter={createHandler($eventMouseEnter, display)}
	on:mouseleave={createHandler($eventMouseLeave, display)}
	on:pointerdown={createPointerDownHandler($_draggable, display, event)}
></div>

<svelte:window on:resize={reposition}/>