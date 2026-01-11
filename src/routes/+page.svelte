<script lang="ts">
	interface SetItem {
		id: number;
		minutes: number;
		seconds: number;
		restMinutes: number;
		restSeconds: number;
		remaining: number;
		isRunning: boolean;
		isResting: boolean;
		isCompleted: boolean;
		intervalId: ReturnType<typeof setInterval> | null;
	}

	interface Exercise {
		id: number;
		name: string;
		sets: SetItem[];
		isCollapsed: boolean;
	}

	let exercises: Exercise[] = $state([]);
	let nextExerciseId = 0;
	let nextSetId = 0;

	function addExercise() {
		exercises.push({
			id: nextExerciseId++,
			name: 'New Exercise',
			sets: [],
			isCollapsed: false
		});
	}

	function removeExercise(id: number) {
		const exercise = exercises.find(e => e.id === id);
		if (exercise) {
			exercise.sets.forEach(set => {
				if (set.intervalId) clearInterval(set.intervalId);
			});
		}
		exercises = exercises.filter(e => e.id !== id);
	}

	function addSet(exercise: Exercise) {
		const last = exercise.sets[exercise.sets.length - 1];
		const minutes = last?.minutes ?? 1;
		const seconds = last?.seconds ?? 0;
		const restMinutes = last?.restMinutes ?? 0;
		const restSeconds = last?.restSeconds ?? 30;
		exercise.sets.push({
			id: nextSetId++,
			minutes,
			seconds,
			restMinutes,
			restSeconds,
			remaining: minutes * 60 + seconds,
			isRunning: false,
			isResting: false,
			isCompleted: false,
			intervalId: null
		});
	}

	function removeSet(exercise: Exercise, setId: number) {
		const set = exercise.sets.find(s => s.id === setId);
		if (set?.intervalId) clearInterval(set.intervalId);
		exercise.sets = exercise.sets.filter(s => s.id !== setId);
	}

	function startTimer(exercise: Exercise, set: SetItem, fresh = false) {
		if (set.isRunning) return;
		if (fresh || set.remaining <= 0) {
			set.remaining = set.minutes * 60 + set.seconds;
		}
		set.isResting = false;
		set.isRunning = true;
		set.intervalId = setInterval(() => {
			if (set.remaining > 0) {
				set.remaining--;
			} else {
				stopTimer(set);
				playAlarm();
				startRest(exercise, set);
			}
		}, 1000);
	}

	function startRest(exercise: Exercise, set: SetItem) {
		const currentIndex = exercise.sets.findIndex(s => s.id === set.id);
		const restTime = set.restMinutes * 60 + set.restSeconds;

		if (currentIndex >= exercise.sets.length - 1 || restTime <= 0) {
			set.isCompleted = true;
			return;
		}

		set.remaining = restTime;
		set.isResting = true;
		set.isRunning = true;
		set.intervalId = setInterval(() => {
			if (set.remaining > 0) {
				set.remaining--;
			} else {
				stopTimer(set);
				set.isCompleted = true;
				playAlarm();
				startNextSet(exercise, set);
			}
		}, 1000);
	}

	function startNextSet(exercise: Exercise, currentSet: SetItem) {
		const currentIndex = exercise.sets.findIndex(s => s.id === currentSet.id);
		if (currentIndex < exercise.sets.length - 1) {
			const nextSet = exercise.sets[currentIndex + 1];
			setTimeout(() => startTimer(exercise, nextSet, true), 500);
		}
	}

	function stopTimer(set: SetItem) {
		if (set.intervalId) {
			clearInterval(set.intervalId);
			set.intervalId = null;
		}
		set.isRunning = false;
	}

	function resetTimer(set: SetItem) {
		stopTimer(set);
		set.isResting = false;
		set.isCompleted = false;
		set.remaining = set.minutes * 60 + set.seconds;
	}

	function formatTime(totalSeconds: number): string {
		const mins = Math.floor(totalSeconds / 60);
		const secs = totalSeconds % 60;
		return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
	}

	function playAlarm() {
		const audioContext = new AudioContext();
		const oscillator = audioContext.createOscillator();
		const gainNode = audioContext.createGain();
		oscillator.connect(gainNode);
		gainNode.connect(audioContext.destination);
		oscillator.frequency.value = 800;
		oscillator.type = 'sine';
		gainNode.gain.setValueAtTime(0.3, audioContext.currentTime);
		gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.5);
		oscillator.start(audioContext.currentTime);
		oscillator.stop(audioContext.currentTime + 0.5);
	}
</script>

<main>
	<h1>Workout Timer</h1>

	<button class="add-btn" onclick={addExercise}>+ Add Exercise</button>

	<div class="exercises-container">
		{#each exercises as exercise (exercise.id)}
			<div class="exercise-card">
				<div class="exercise-header">
					<input
						type="text"
						class="exercise-name"
						value={exercise.name}
						oninput={(e) => { exercise.name = e.currentTarget.value; }}
					/>
					<div class="exercise-actions">
						<button class="collapse-btn" onclick={() => { exercise.isCollapsed = !exercise.isCollapsed; }}>
							{exercise.isCollapsed ? '+' : '-'}
						</button>
						<button class="delete-btn" onclick={() => removeExercise(exercise.id)}>x</button>
					</div>
				</div>

				{#if !exercise.isCollapsed}
					<div class="sets-container">
						{#each exercise.sets as set, index (set.id)}
							<div class="set-card" class:running={set.isRunning && !set.isResting} class:resting={set.isResting}>
								<div class="set-header">
									<span class="set-number">Set {index + 1}</span>
									{#if set.isCompleted}
										<span class="complete-badge">COMPLETE</span>
									{:else if set.isResting}
										<span class="rest-badge">REST</span>
									{/if}
									<button class="delete-btn small" onclick={() => removeSet(exercise, set.id)}>x</button>
								</div>

								<div class="timer-display">
									{formatTime(set.remaining)}
								</div>

								<div class="time-inputs">
									<div class="input-group">
										<span class="input-label">Work</span>
										<div class="input-row">
											<label>
												<input
													type="number"
													value={set.minutes}
													oninput={(e) => { set.minutes = +e.currentTarget.value; set.remaining = set.minutes * 60 + set.seconds; }}
													min="0"
													max="99"
													disabled={set.isRunning}
												/>
												<span>min</span>
											</label>
											<label>
												<input
													type="number"
													value={set.seconds}
													oninput={(e) => { set.seconds = +e.currentTarget.value; set.remaining = set.minutes * 60 + set.seconds; }}
													min="0"
													max="59"
													disabled={set.isRunning}
												/>
												<span>sec</span>
											</label>
										</div>
									</div>
									<div class="input-group">
										<span class="input-label">Rest</span>
										<div class="input-row">
											<label>
												<input
													type="number"
													value={set.restMinutes}
													oninput={(e) => { set.restMinutes = +e.currentTarget.value; }}
													min="0"
													max="99"
													disabled={set.isRunning}
												/>
												<span>min</span>
											</label>
											<label>
												<input
													type="number"
													value={set.restSeconds}
													oninput={(e) => { set.restSeconds = +e.currentTarget.value; }}
													min="0"
													max="59"
													disabled={set.isRunning}
												/>
												<span>sec</span>
											</label>
										</div>
									</div>
								</div>

								<div class="controls">
									{#if set.isRunning}
										<button class="control-btn stop" onclick={() => stopTimer(set)}>Stop</button>
									{:else}
										<button class="control-btn start" onclick={() => startTimer(exercise, set)}>Start</button>
									{/if}
									<button class="control-btn reset" onclick={() => resetTimer(set)}>Reset</button>
								</div>
							</div>
						{/each}
					</div>

					<button class="add-set-btn" onclick={() => addSet(exercise)}>+ Add Set</button>
				{/if}
			</div>
		{/each}
	</div>

	{#if exercises.length === 0}
		<p class="empty-message">Click "+ Add Exercise" to start</p>
	{/if}
</main>

<style>
	:global(body) {
		margin: 0;
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
		background: #1a1a2e;
		color: #eee;
		min-height: 100vh;
	}

	main {
		max-width: 600px;
		margin: 0 auto;
		padding: 2rem;
	}

	h1 {
		text-align: center;
		color: #fff;
		margin-bottom: 2rem;
	}

	.add-btn {
		display: block;
		width: 100%;
		padding: 1rem;
		font-size: 1.2rem;
		background: #4a47a3;
		color: white;
		border: none;
		border-radius: 12px;
		cursor: pointer;
		transition: background 0.2s;
	}

	.add-btn:hover {
		background: #5c59b0;
	}

	.exercises-container {
		margin-top: 1.5rem;
		display: flex;
		flex-direction: column;
		gap: 1.5rem;
	}

	.exercise-card {
		background: #0f1729;
		border-radius: 20px;
		padding: 1.5rem;
		border: 1px solid #2a3a5a;
	}

	.exercise-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 1rem;
	}

	.exercise-name {
		background: transparent;
		border: none;
		border-bottom: 2px solid #4a47a3;
		color: #fff;
		font-size: 1.3rem;
		font-weight: 700;
		padding: 0.25rem 0;
		flex: 1;
		margin-right: 1rem;
	}

	.exercise-name:focus {
		outline: none;
		border-bottom-color: #6c69d1;
	}

	.exercise-actions {
		display: flex;
		gap: 0.5rem;
	}

	.collapse-btn {
		background: #3b4a6b;
		color: white;
		border: none;
		width: 28px;
		height: 28px;
		border-radius: 6px;
		cursor: pointer;
		font-size: 1.2rem;
		font-weight: 700;
	}

	.sets-container {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.set-card {
		background: #16213e;
		border-radius: 16px;
		padding: 1.25rem;
		border: 2px solid transparent;
		transition: border-color 0.3s, background 0.3s;
	}

	.set-card.running {
		border-color: #4ade80;
	}

	.set-card.resting {
		border-color: #60a5fa;
		background: #1e3a5f;
	}

	.set-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 0.75rem;
	}

	.set-number {
		font-weight: 600;
		color: #a5b4fc;
	}

	.rest-badge {
		background: #3b82f6;
		color: white;
		padding: 0.25rem 0.75rem;
		border-radius: 20px;
		font-size: 0.75rem;
		font-weight: 700;
	}

	.complete-badge {
		background: #22c55e;
		color: white;
		padding: 0.25rem 0.75rem;
		border-radius: 20px;
		font-size: 0.75rem;
		font-weight: 700;
	}

	.delete-btn {
		background: #ef4444;
		color: white;
		border: none;
		width: 28px;
		height: 28px;
		border-radius: 50%;
		cursor: pointer;
		font-size: 1rem;
	}

	.delete-btn.small {
		width: 24px;
		height: 24px;
		font-size: 0.85rem;
	}

	.timer-display {
		font-size: 3rem;
		font-weight: 700;
		text-align: center;
		font-family: 'Courier New', monospace;
		color: #fff;
		margin: 0.75rem 0;
	}

	.time-inputs {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		margin-bottom: 0.75rem;
	}

	.input-group {
		display: flex;
		align-items: center;
		gap: 1rem;
	}

	.input-label {
		width: 40px;
		font-size: 0.85rem;
		color: #888;
		font-weight: 600;
	}

	.input-row {
		display: flex;
		gap: 1rem;
	}

	.input-row label {
		display: flex;
		align-items: center;
		gap: 0.5rem;
	}

	.input-row input {
		width: 50px;
		padding: 0.5rem;
		font-size: 1rem;
		border: none;
		border-radius: 8px;
		background: #0f3460;
		color: #fff;
		text-align: center;
	}

	.input-row input:disabled {
		opacity: 0.5;
	}

	.input-row span {
		color: #888;
		font-size: 0.85rem;
	}

	.controls {
		display: flex;
		gap: 0.5rem;
	}

	.control-btn {
		flex: 1;
		padding: 0.75rem;
		font-size: 1rem;
		border: none;
		border-radius: 8px;
		cursor: pointer;
		font-weight: 600;
		transition: opacity 0.2s;
	}

	.control-btn:hover {
		opacity: 0.9;
	}

	.control-btn.start {
		background: #22c55e;
		color: white;
	}

	.control-btn.stop {
		background: #ef4444;
		color: white;
	}

	.control-btn.reset {
		background: #6b7280;
		color: white;
	}

	.add-set-btn {
		width: 100%;
		padding: 0.75rem;
		margin-top: 1rem;
		font-size: 1rem;
		background: #2a3a5a;
		color: #a5b4fc;
		border: 2px dashed #4a5a7a;
		border-radius: 12px;
		cursor: pointer;
		transition: background 0.2s;
	}

	.add-set-btn:hover {
		background: #3a4a6a;
	}

	.empty-message {
		text-align: center;
		color: #666;
		margin-top: 3rem;
		font-size: 1.1rem;
	}

	/* 모바일 반응형 스타일 */
	@media (max-width: 480px) {
		main {
			padding: 1rem;
		}

		h1 {
			font-size: 1.5rem;
		}

		.exercise-card {
			padding: 1rem;
			overflow: hidden;
		}

		.exercise-header {
			gap: 0.5rem;
		}

		.exercise-name {
			font-size: 1.1rem;
			min-width: 0;
			margin-right: 0.5rem;
		}

		.exercise-actions {
			flex-shrink: 0;
		}

		.set-card {
			padding: 1rem;
			overflow: hidden;
		}

		.timer-display {
			font-size: 2.25rem;
		}

		.input-group {
			flex-direction: column;
			align-items: flex-start;
			gap: 0.25rem;
		}

		.input-label {
			width: auto;
		}

		.input-row {
			width: 100%;
			gap: 0.5rem;
		}

		.input-row label {
			flex: 1;
			gap: 0.25rem;
		}

		.input-row input {
			width: 100%;
			min-width: 40px;
			padding: 0.4rem;
			font-size: 0.9rem;
		}

		.input-row span {
			font-size: 0.75rem;
			white-space: nowrap;
		}

		.control-btn {
			padding: 0.6rem;
			font-size: 0.9rem;
		}
	}
</style>
