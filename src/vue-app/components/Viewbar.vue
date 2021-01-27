<template>
	<div class="viewbar">
		<div>
			<div
				:class="viewbarEntryClass(Modes.NOTES)"
				@keydown="evt => setMode(Modes.NOTES, evt)"
				@click="setMode(Modes.NOTES)"
				v-tippy="{ placement: 'right', animateFill: false, delay: 300 }"
				title="Write"
				content="Write"
				tabindex="0"
			>
				<svg class="viewbar__icon" fill="currentColor" viewBox="0 0 20 20"><path d="M17.414 2.586a2 2 0 00-2.828 0L7 10.172V13h2.828l7.586-7.586a2 2 0 000-2.828z"></path><path fill-rule="evenodd" d="M2 6a2 2 0 012-2h4a1 1 0 010 2H4v10h10v-4a1 1 0 112 0v4a2 2 0 01-2 2H4a2 2 0 01-2-2V6z" clip-rule="evenodd"></path></svg>
			</div>

			<div
				:class="viewbarEntryClass(Modes.CATEGORIES)"
				@keydown="evt => setMode(Modes.CATEGORIES, evt)"
				@click="setMode(Modes.CATEGORIES)"
				v-tippy="{ placement: 'right', animateFill: false, delay: 300 }"
				title="Projects"
				content="Projects"
				tabindex="0"
			>
				<svg class="viewbar__icon" fill="currentColor" viewBox="0 0 640 512"><path d="M592 64H400L345.37 9.37c-6-6-14.14-9.37-22.63-9.37H176c-26.51 0-48 21.49-48 48v80H48c-26.51 0-48 21.49-48 48v288c0 26.51 21.49 48 48 48h416c26.51 0 48-21.49 48-48v-80h80c26.51 0 48-21.49 48-48V112c0-26.51-21.49-48-48-48zM464 464H48V176h80v160c0 26.51 21.49 48 48 48h288v80zm128-128H176V48h140.12l54.63 54.63c6 6 14.14 9.37 22.63 9.37H592v224z"/></svg>
			</div>

			<div :class="viewbarEntryClass(Modes.ARCHIVE)"
				@keydown="evt => setMode(Modes.ARCHIVE, evt)"
				@click="setMode(Modes.ARCHIVE)"
				v-tippy="{ placement: 'right', animateFill: false, delay: 300 }"
				title="Archive"
				content="Archive"
				tabindex="0"
			>
				<svg class="viewbar__icon" fill="currentColor" viewBox="0 0 512 512"><path d="M464 32H48C21.5 32 0 53.5 0 80v80c0 8.8 7.2 16 16 16h16v272c0 17.7 14.3 32 32 32h384c17.7 0 32-14.3 32-32V176h16c8.8 0 16-7.2 16-16V80c0-26.5-21.5-48-48-48zm-32 400H80V176h352v256zm32-304H48V80h416v48zM204 272h104c6.6 0 12-5.4 12-12v-24c0-6.6-5.4-12-12-12H204c-6.6 0-12 5.4-12 12v24c0 6.6 5.4 12 12 12z"/></svg>
			</div>

			<div class="viewbar__projects" v-if="pinnedProjects.length">
				<div
					v-for="project in pinnedProjects"
					v-tippy="{ placement: 'right', animateFill: false, delay: 300 }"
					class="viewbar__entry"
					@keydown="evt => setActiveProject(project.tag, evt)"
					@click="setActiveProject(project.tag)"
					:title="project.title"
					:content="project.title"
					:key="project.tag"
					tabindex="0"
				>
					<svg class="viewbar__icon" fill="currentColor" viewBox="0 0 512 512"><path d="M464 128H272l-54.63-54.63c-6-6-14.14-9.37-22.63-9.37H48C21.49 64 0 85.49 0 112v288c0 26.51 21.49 48 48 48h416c26.51 0 48-21.49 48-48V176c0-26.51-21.49-48-48-48zm0 272H48V112h140.12l54.63 54.63c6 6 14.14 9.37 22.63 9.37H464v224z"/></svg>
				</div>
			</div>
		</div>
	</div>
</template>

<script>
/* eslint-disable indent */
	import Modes from "@/store/modes";
	import Actions from "@/store/actions";

	import { isActionableKey } from "@/utilities/keycode";

	export default {
		data() {
			return {
				Modes,
			};
		},
		computed: {
			currentMode() {
				return this.$store.state.mode;
			},
			pinnedProjects() {
				return Object.values(this.$store.state.projects)
					.filter(project => project.pinned);
			}
		},
		methods: {
			setMode(mode, event) {
				if (event && !isActionableKey(event)) {
					return;
				}

				return this.$store.commit(Actions.SET_MODE, mode);
			},

			setActiveProject(tag, event) {
				if (event && !isActionableKey(event)) {
					return;
				}

				this.$store.commit(Actions.SET_ACTIVE_PROJECT, tag);
				return this.$store.commit(Actions.SET_MODE, Modes.CATEGORIES);
			},

			addNote(event) {
				if (event && !isActionableKey(event)) {
					return;
				}

				return this.$store.dispatch(Actions.ADD_NOTE);
			},

			isNoteActive(id) {
				return this.$store.state.activeNote === id;
			},

			viewbarEntryClass(mode) {
				return {
					"viewbar__entry": true,
					"viewbar--notes": mode === Modes.NOTES,
					"viewbar--categories": mode === Modes.CATEGORIES,
					"viewbar--projects": mode === Modes.PROJECTS,
					"is-active": this.currentMode === mode,
				};
			},
		}
	};
</script>


<style lang="scss">
	.viewbar {
		display: flex;
		flex-direction: row;
		background: var(--muted);
		padding: .33rem 1rem;
		justify-content: space-between;

		&__entry {
			display: inline-block;
			padding: .66rem;
			color: var(--text--dark);
			margin: .1666667rem 0;
			border-radius: 4px;
			max-height: 40px;

			&:hover,
			&:focus {
				cursor: pointer;
				background: var(--muted--intense);
				outline: none;
				color: var(--text);

				&.is-active {
					background: var(--primary--light);
					color: var(--primary--inverse);
				}
			}

			&:first-child { margin-top: 0; }
			&:last-child { margin-bottom: 0; }

			&.is-active {
				cursor: pointer;
				background: var(--muted--intense);
				outline: none;
				color: var(--text);

			}

			&.viewbar--add {
				color: var(--primary);
			}
		}

		&__icon {
			line-height: 1;
			width: 20px;
			height: 20px;
		}

		&__projects {
			margin-top: 1rem;
			padding-top: 1rem;
			border-top: 1px solid var(--muted--intense);
		}
	}
</style>
