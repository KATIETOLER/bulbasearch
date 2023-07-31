<script lang="ts">
	import Pokemon from './pokemon.svelte'
	import Icon from '$lib/assets/pokeball.png'
	import { onMount } from 'svelte'
	import { of, fromEvent, Observable, throwError, EMPTY } from 'rxjs'
	import { fromFetch } from 'rxjs/fetch'
	import {
		map,
		catchError,
		switchMap,
		startWith,
		debounceTime,
		distinctUntilChanged,
		delay,
		tap,
		filter,
		retry,
		share,
		expand,
		reduce,
		scan,
	} from 'rxjs/operators'

	// UI states
	// This enum represents our finite list of possible states
	enum State {
		unrequested,
		emptyResults,
		loading,
		error,
		fetchedResults,
	}

	// Mock response to model data so I don't overload the API
	const mockResponse = (query: string) => {
		return of(
			new Response(
				JSON.stringify({
					query,
					'pokemon': [
						{ 'id': 1, 'name': 'Bulbasaur', 'classfication': 'Seed Pokémon' },
					],
					'nextPage': 'next-page-token',
				})
			)
		).pipe(delay(375))
	}

	// Created type Pokemon with `classification` spelling error to model API
	type Pokemon = {
		id: string
		name: string
		classfication: string
	}

	type ApiResponse = {
		pokemon: Pokemon[]
		nextPage: string
	}

	let shouldResetResults = true
	let queryString: string = ''
	let state: State = State.unrequested
	let searchElement: HTMLInputElement
	let pokemonStream: Observable<Pokemon[]>

	onMount(async () => {
		let searchStream = fromEvent(searchElement, 'input').pipe(
			debounceTime(375),
			// @ts-ignore
			// Not sure why this type isn't being accepted
			map((e) => e.target?.value.toLowerCase() as string),
			distinctUntilChanged(),
			// Operator that keeps the fetch from firing until the input is different
			// if a user copy-pastes the same info the fetch will not fire
			filter((val) => val.trim().length > 0)
			// Making sure only events that are not empty go through
		)

		let apiStream = searchStream.pipe(
			tap((query) => {
				// Escape hatch to switch state status
				// Side effect (code that executes and modifies non-local state)
				// This changes the state to 'loading' while we're searching
				state = State.loading

				// Save the query string for later so we can handle pagination
				queryString = query

				// Make sure we reset the results so we don't show the results of multiple
				// search queries. I don't really like this solution because it's using
				// some random state I've set. I would like to have a solution handles this
				// using just streams but time did not permit that.
				shouldResetResults = true
			}),
			switchMap((query) => {
				// Query with switchMap ensures the most recent event is the only response from the query
				return fromFetch(
					`https://hungry-woolly-leech.glitch.me/api/pokemon/search/${query}`
				)
				// return mockResponse(query)
			}),
			// Operator that resends network request twice if error (unsubscribes and resubscribes)
			retry(2)
		)

		// Pagination using RxJS. Had to do some research on this since I've never handled pagination
		// in RxJS. It was tricky and this part I completed after the two hours but I wanted to see
		// a solution through to completion. I used these links as starting points:
		// https://stackoverflow.com/questions/35254323/rxjs-observable-pagination
		// https://www.learnrxjs.io/learn-rxjs/operators/transformation/expand
		// https://nightwolf.dev/get-all-paginated-data-using-rxjs-expand-operator/#0
		let responseStream = apiStream.pipe(
			switchMap((res) => {
				if (res.ok) {
					return res.json() // json() is a promise
				} else {
					return throwError(() => new Error(res.statusText))
				}
			}),
			map((res) => res as ApiResponse),
			expand((res) => {
				if (res.nextPage) {
					console.log('fetching next page...')
					// This is duplicated logic that we can reuse maybe later?
					return fromFetch(
						`https://hungry-woolly-leech.glitch.me/api/pokemon/search/${queryString}?page=${res.nextPage}`
					).pipe(
						retry(2),
						switchMap((res) => {
							if (res.ok) {
								return res.json()
							} else {
								// Should we error here? Should we just return nothing?
								return of({ pokemon: [], nextPage: '' })
							}
						}),
						delay(500) // don't spam the api
					)
				} else {
					return EMPTY
				}
			}),

			// For every new page of pokemon, we are adding to the current list of pokemon
			// Definitely looked up scan. Also haven't used concat in a long time.
			// https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/concat
			// https://www.learnrxjs.io/learn-rxjs/operators/transformation/scan

			scan((acc: Pokemon[], val: ApiResponse) => {
				// This logic resets the results whenever a new search is started. I'd like to
				// have a better solution for this (see earlier comments).
				if (shouldResetResults) {
					shouldResetResults = false
					return val.pokemon
				} else {
					return acc.concat(val.pokemon)
				}
			}, []),

			// Set the app's UI state
			tap((res) => {
				if (res.length === 0) {
					state = State.emptyResults
				} else {
					state = State.fetchedResults
				}
			})
		)

		// Finally catch any errors and set the UI error state.
		// Start with an empty array just to be safe.
		pokemonStream = responseStream.pipe(
			catchError((err) => {
				state = State.error
				console.log(err)
				return of([])
			}),
			startWith([])
		)
	})
</script>

<div class="search-container">
	<div class="search-outline">
		<input
			class="search-input"
			bind:this={searchElement}
			placeholder="Search pokedex..."
		/>
		<img src={Icon} alt="pokeball search icon" id="searchIcon" />
	</div>
	<!-- Should this logic live elsewhere?  -->
	{#if state === State.unrequested}
		<p>
			<em> unrequested </em>
		</p>
	{:else if state === State.loading}
		<p>Loading...</p>
	{:else if state === State.emptyResults}
		<p>No results.</p>
	{:else if state === State.error}
		<p>A wild error appeared! Sorry, something went wrong; try again later.</p>
	{:else if state === State.fetchedResults}
		{#each $pokemonStream as pokemon}
			<Pokemon name={pokemon.name} classfication={pokemon.classfication} />
		{/each}
	{/if}
</div>

<style>
	.search-container {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		padding: 10em;
	}

	#searchIcon {
		max-height: 2em;
		padding: 0.3em;
	}

	.search-outline {
		display: flex;
		background-color: rgb(253, 237, 65);
		padding: 0.5em;
		padding-right: 0.3em;
		border-style: solid;
		border-radius: 0.5em;
		border-color: gray;
	}

	.search-input {
		padding: 1em;
	}
</style>
