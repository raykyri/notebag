<template>
	<div :class="overviewClass(isArchive)">
		<input v-model="searchInput" @keydown="handleEnter" ref="searchField" class="note-overview__search" placeholder="Search…" type="text" tabindex="0"/>
		<div class="note-overview__notes">
			<note-list ref="list" :archive="isArchive" :notes="searchResults"></note-list>
		</div>
	</div>
</template>

<script>
/* eslint-disable indent */
import {isEnterKey} from "@/utilities/keycode";

const FUSE_OPTIONS = {
	isCaseSensitive: false,
	includeMatches: false,
	includeScore: false,
	useExtendedSearch: true,
	findAllMatches: false,
	minMatchCharLength: 5,
	shouldSort: true,
	threshold: 0.6,
	location: 0,
	distance: 10000,
	keys: [
		"title",
		"preview",
		"categories",
	],
};

	import { mapState, mapGetters } from "vuex";
	import Fuse from "fuse.js";
	import Mousetrap from "mousetrap";

	import EventBus, { Events } from "@/EventBus";
	import Actions from "@/store/actions";
	import Modes from "@/store/modes";

	import NoteList from "@/components/NoteList";

	export default {
		name: "NoteOverview",
		components: {NoteList},
		watch: {
			notes: {
				handler(newNotes) {
					const notes = Object.values(newNotes);

					this.searchAgent = new Fuse(notes, FUSE_OPTIONS);
				},
				deep: true,
			}
		},
		computed: {
			...mapGetters({
				notes: "sortedNotes"
			}),
			...mapState(["mode"]),
			isArchive() {
				return this.mode === Modes.ARCHIVE;
			},
			searchResults() {
				if (!this.mode === Modes.NOTES) { console.log(this.notes.length); return this.notes; }
				if (!this.searchInput && this.mode === Modes.NOTES) { console.log(this.notes.length); return this.notes; }
				if (!this.searchInput && this.isArchive) { console.log("archive"); return this.$store.state.archivedNotes; }

				const results = this.searchAgent.search(this.searchInput).map(result => result.item);
				console.log(results.length);
				return results;
			},
		},
		events: {
			[Events.MODE_SWITCHED_AFTER]: {
				priority: 0,
				handler(event, mode) {
					if (mode === Modes.EDITOR || mode === Modes.ARCHIVE) {
						this.setupNotesTab();
					}
				}
			},

			[Events.MODE_SWITCHED]: {
				priority: 10,
				handler(event, mode) {
					if (mode !== Modes.EDITOR && mode !== Modes.ARCHIVE) {
						this.teardownNotesTab();
					}
				}
			},
		},
		methods: {
			async handleEnter(event) {
				if (isEnterKey(event)) {
					const text = event.target.value;
					if (text.trim() === "") return;

					const [ note ] = this.searchResults;
					if (note && this.$store.state.activeNote === note.id) {
						this.$store.commit(Actions.SET_ACTIVE_NOTE, note.id);
					} else {
						const result = await this.$store.dispatch(Actions.ADD_NOTE, { title: text.trim() });
						this.$store.commit(Actions.SET_ACTIVE_NOTE, result.id);
						// TODO: focus edito
					}
				}
			},
			cycleElements(event) {
				this.$nextTick(() => {
					const tabbableElements = document.querySelectorAll(".note-overview [tabindex]:not(.no-cycle)");

					let operand = event.which === 40 ? 1 : -1;
					this.selectedIndex = this.selectedIndex + operand;

					if (this.selectedIndex > tabbableElements.length - 1) {
						this.selectedIndex = 0;
					}

					if (this.selectedIndex < 0) {
						this.selectedIndex = tabbableElements.length - 1;
					}

					tabbableElements[this.selectedIndex].focus();
				});
			},
			switchToNote(idx) {
				const notes = Object.values(this.searchResults);
				const note = notes[idx];

				if (note) {
					this.$store.commit(Actions.SET_ACTIVE_NOTE, note.id);
					// this.$nextTick(() => {
					// 	document.querySelector("#note-" + note.id).scrollIntoView({ smooth: true, block: "nearest" });
					// });
				}
			},
			selectPrevNote() { console.log("but...", this.searchResults.length);
				const notes = Object.values(this.searchResults);
				const idx = notes.findIndex((note) => note.id === this.$store.state.activeNote);
				const note = idx === -1 ? notes[0] : notes[idx - 1];

				if (note) {
					this.$store.commit(Actions.SET_ACTIVE_NOTE, note.id);
					this.$nextTick(() => {
						document.querySelector("#note-" + note.id).scrollIntoView({ smooth: true, block: "nearest" });
					});
				}
			},
			selectNextNote() {
				const notes = Object.values(this.searchResults);
				const idx = notes.findIndex((note) => note.id === this.$store.state.activeNote);
				const note = idx === -1 ? notes[0] : notes[idx + 1];

				if (note) {
					this.$store.commit(Actions.SET_ACTIVE_NOTE, note.id);
					this.$nextTick(() => {
						document.querySelector("#note-" + note.id).scrollIntoView({ smooth: true, block: "nearest" });
					});
				}
			},
			setupNotesTab() {
				this.selectedIndex = 0;
				this.$nextTick(() => this.$refs.searchField.focus());

				for (let i=1; i<10; i++) {
					Mousetrap.bind(`ctrl+${i}`, () => this.switchToNote(i-1));
				}

				Mousetrap.bind("down", this.cycleElements);
				Mousetrap.bind("up", this.cycleElements);
			},
			teardownNotesTab() {
				for (let i=1; i<10; i++) {
					Mousetrap.unbind(`ctrl+${i}`);
				}

				Mousetrap.unbind("down");
				Mousetrap.unbind("up");
			},
			overviewClass(isArchive) {
				return {
					"note-overview": true,
					"note-overview--archive": isArchive,
					"note-overview--regular": !isArchive,
				};
			}
		},
		data() {
			const notes = this.isArchive
				? Object.values(this.$store.state.archivedNotes)
				: Object.values(this.$store.getters.sortedNotes);

			return {
				mode: Modes.NOTES,
				searchInput: "",
				searchAgent: new Fuse(notes, FUSE_OPTIONS),
				selectedIndex: 0,
			};
		},
		created() {
			// remove any previous listeners
			EventBus.$off(Events.SEARCHING_CATEGORY);
			EventBus.$off(Events.REQUESTING_PREV_NOTE);
			EventBus.$off(Events.REQUESTING_NEXT_NOTE);

			// add fresh listeners
			EventBus.$on(Events.SEARCHING_CATEGORY, category => this.searchInput = category);
			EventBus.$on(Events.REQUESTING_PREV_NOTE, () => this.selectPrevNote());
			EventBus.$on(Events.REQUESTING_NEXT_NOTE, () => this.selectNextNote());
		},
	};
</script>

<style lang="scss">
	.note-overview {
		width: 100%;
		height: 18rem;
		padding: 0.8rem 0.8rem 0.2rem;
		position: relative;
		background-color: var(--note-pane);
		overflow-y: auto;

		&__title {
			margin-top: 0;
			user-select: none;
		}

		&__search {
			color: var(--text);
			background: var(--muted);
			border-radius: 4px;
			padding: .55rem .66rem;
			width: 100%;
			outline: none;
			border: none;
		}

		&__notes {
			margin-top: 0.25rem;
		}
	}

	.note-overview::-webkit-scrollbar {
		display: none;
	}
</style>
