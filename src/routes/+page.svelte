<script>
	import Icon from '@iconify/svelte';
	import { onMount } from 'svelte';

	function getLolTier(skill) {
		if (skill <= 9) return { name: '아이언', icon: 'mdi:medal', color: 'text-gray-500' };
		if (skill <= 19) return { name: '브론즈', icon: 'mdi:medal', color: 'text-yellow-700' };
		if (skill <= 29) return { name: '실버', icon: 'mdi:medal', color: 'text-gray-400' };
		if (skill <= 39) return { name: '골드', icon: 'mdi:medal', color: 'text-yellow-400' };
		if (skill <= 49) return { name: '플래티넘', icon: 'mdi:medal', color: 'text-cyan-400' };
		if (skill <= 59) return { name: '에메랄드', icon: 'mdi:medal', color: 'text-emerald-400' };
		if (skill <= 69) return { name: '다이아', icon: 'mdi:medal', color: 'text-blue-400' };
		if (skill <= 79) return { name: '마스터', icon: 'mdi:crown-outline', color: 'text-purple-500' };
		if (skill <= 89) return { name: '그랜드마스터', icon: 'mdi:crown', color: 'text-red-500' };
		return { name: '챌린저', icon: 'mdi:star-check-outline', color: 'text-yellow-500' };
	}

	let people = $state([]);
	let loading = $state(false);
	let newPerson = $state('');
	let editingSkillId = $state(null);
	let tempSkill = $state('');
	let waitingList = $state([]);
	let simulationCount = $state(10);
	let teamAssignments = $state({
		A: [null, null, null, null],
		B: [null, null, null, null]
	});

	// 모달 상태
	let modalOpen = $state(false);
	let selectedTeam = $state('');
	let selectedPosition = $state(0);

	// Derived values
	const teamACount = $derived(teamAssignments.A.filter((p) => p !== null).length);
	const teamBCount = $derived(teamAssignments.B.filter((p) => p !== null).length);
	const maxAssigned = 8;
	const allAssigned = $derived(
		[...teamAssignments.A, ...teamAssignments.B].filter((p) => p !== null)
	);
	const uniqueAssignedCount = $derived(Array.from(new Set(allAssigned.map((p) => p?.name))).length);
	const unassignedCount = $derived(maxAssigned - uniqueAssignedCount);
	const canAddPerson = $derived(
		newPerson.trim() && !people.some((p) => p.name === newPerson.trim()) && people.length < 20
	);

	// 팀별 점수 총합
	const teamASum = $derived(teamAssignments.A.reduce((sum, p) => sum + (p?.skill ?? 0), 0));
	const teamBSum = $derived(teamAssignments.B.reduce((sum, p) => sum + (p?.skill ?? 0), 0));

	// 티어 분포 계산
	const tierDistribution = $derived.by(() => {
		const distribution = {
			아이언: { count: 0, range: '0-9', color: 'text-gray-500' },
			브론즈: { count: 0, range: '10-19', color: 'text-yellow-700' },
			실버: { count: 0, range: '20-29', color: 'text-gray-400' },
			골드: { count: 0, range: '30-39', color: 'text-yellow-400' },
			플래티넘: { count: 0, range: '40-49', color: 'text-cyan-400' },
			에메랄드: { count: 0, range: '50-59', color: 'text-emerald-400' },
			다이아: { count: 0, range: '60-69', color: 'text-blue-400' },
			마스터: { count: 0, range: '70-79', color: 'text-purple-500' },
			그랜드마스터: { count: 0, range: '80-89', color: 'text-red-500' },
			챌린저: { count: 0, range: '90-100', color: 'text-yellow-500' }
		};

		people.forEach((person) => {
			const tier = getLolTier(person.skill);
			if (distribution[tier.name]) {
				distribution[tier.name].count++;
			}
		});

		return distribution;
	});

	// 로컬스토리지 함수들
	function savePeopleToLocalStorage() {
		localStorage.setItem('team-matching-people', JSON.stringify(people));
	}

	function loadPeopleFromLocalStorage() {
		const stored = localStorage.getItem('team-matching-people');
		if (stored) {
			try {
				people = JSON.parse(stored);
			} catch (e) {
				console.error('로컬스토리지 데이터 파싱 오류:', e);
				people = [];
			}
		}
		loading = false;
	}

	function updateSkill(person, newSkill) {
		const personIndex = people.findIndex((p) => p.id === person.id);
		if (personIndex !== -1) {
			people[personIndex].skill = newSkill;
			person.skill = newSkill;
			savePeopleToLocalStorage();
		}
	}

	onMount(() => {
		loadPeopleFromLocalStorage();
	});

	function addPerson() {
		if (canAddPerson) {
			const newPersonData = {
				id: Date.now(), // 간단한 ID 생성
				name: newPerson.trim(),
				skill: 0,
				created_at: new Date().toISOString()
			};
			people.push(newPersonData);
			savePeopleToLocalStorage();
			newPerson = '';
		}
	}

	function removePerson(personToRemove) {
		// 팀 배정에서 제거
		teamAssignments.A = teamAssignments.A.map((p) => (p?.name === personToRemove.name ? null : p));
		teamAssignments.B = teamAssignments.B.map((p) => (p?.name === personToRemove.name ? null : p));

		// 대기 리스트에서 제거
		waitingList = waitingList.filter((p) => p.id !== personToRemove.id);

		// 인원 목록에서 제거
		people = people.filter((p) => p.id !== personToRemove.id);
		savePeopleToLocalStorage();
	}

	function availableOptions(team, idx) {
		return people.filter(
			(p) =>
				!allAssigned.some((ap) => ap?.name === p.name) ||
				teamAssignments[team][idx]?.name === p.name
		);
	}

	function openModal(team, position) {
		selectedTeam = team;
		selectedPosition = position;
		modalOpen = true;
	}

	function selectPerson(person) {
		teamAssignments[selectedTeam][selectedPosition] = person;
		modalOpen = false;
	}

	function clearAll() {
		teamAssignments.A = [null, null, null, null];
		teamAssignments.B = [null, null, null, null];
	}

	function addToWaitingList(person) {
		if (!waitingList.some((p) => p.id === person.id)) {
			waitingList = [...waitingList, person];
		}
	}

	function removeFromWaitingList(person) {
		waitingList = waitingList.filter((p) => p.id !== person.id);
	}

	function shuffleWaitingList() {
		if (waitingList.length === 0) return;

		// 팀 초기화
		teamAssignments.A = [null, null, null, null];
		teamAssignments.B = [null, null, null, null];

		const toAssign = [...waitingList];
		if (toAssign.length < 2) return;

		let bestAssignment = null;
		let smallestDifference = Infinity;

		for (let i = 0; i < simulationCount; i++) {
			const shuffled = [...toAssign].sort(() => Math.random() - 0.5);
			const totalPlayers = Math.min(shuffled.length, 8);
			const teamSize = Math.floor(totalPlayers / 2);
			const teamA = shuffled.slice(0, teamSize);
			const teamB = shuffled.slice(teamSize, teamSize * 2);

			teamA.sort((a, b) => b.skill - a.skill);
			teamB.sort((a, b) => b.skill - a.skill);

			const teamAScore = teamA.reduce((sum, person) => sum + person.skill, 0);
			const teamBScore = teamB.reduce((sum, person) => sum + person.skill, 0);
			const difference = Math.abs(teamAScore - teamBScore);

			if (difference < smallestDifference) {
				smallestDifference = difference;
				bestAssignment = { teamA: [...teamA], teamB: [...teamB] };
			}
		}

		if (bestAssignment) {
			bestAssignment.teamA.forEach((person, index) => {
				if (index < 4) {
					teamAssignments.A[index] = person;
				}
			});

			bestAssignment.teamB.forEach((person, index) => {
				if (index < 4) {
					teamAssignments.B[index] = person;
				}
			});
		}
	}

	function handleKeydown(event) {
		if (event.key === 'Enter') {
			addPerson();
		}
	}
</script>

<div class="min-h-screen bg-base-200">
	<!-- Header -->
	<div class="">
		<div class="mx-auto max-w-7xl px-4 py-6">
			<h1
				class="mb-6 flex items-center justify-center text-center text-4xl font-bold text-base-content"
			>
				<img src="./logo.svg" alt="logo" class="size-40" />타족단 팀 매칭
			</h1>
			<!-- 통계 정보 -->
			<div class="stats flex stats-horizontal justify-center bg-base-200 shadow-lg">
				<div class="stat place-items-center">
					<div class="stat-title">총 인원</div>
					<div class="stat-value text-lg">{people.length}</div>
				</div>
				<div class="stat place-items-center">
					<div class="stat-title">A팀</div>
					<div class="stat-value text-lg text-primary">{teamACount}/4</div>
				</div>
				<div class="stat place-items-center">
					<div class="stat-title">B팀</div>
					<div class="stat-value text-lg text-secondary">{teamBCount}/4</div>
				</div>
				<div class="stat place-items-center">
					<div class="stat-title">미배정</div>
					<div class="stat-value text-lg text-warning">{unassignedCount}</div>
				</div>
			</div>
		</div>
	</div>

	<!-- Main Content -->
	<div class="mx-auto max-w-7xl px-6 py-8">
		<div class="grid grid-cols-1 gap-8 md:grid-cols-7">
			<div class="col-span-2">
				<!-- 인원 관리 -->
				<div class="card flex h-full flex-1 flex-col bg-base-100 shadow-xl" id="main-card">
					<div class="card-body flex h-full flex-col p-6">
						<h2 class="card-title text-lg">👥 인원 관리</h2>
						<!-- 추가 -->
						<div class="mb-4 flex gap-2">
							<input
								class="input-bordered input input-sm flex-1"
								type="text"
								placeholder="이름 입력"
								bind:value={newPerson}
								onkeydown={handleKeydown}
							/>
							<button class="btn btn-sm btn-success" onclick={addPerson} disabled={!canAddPerson}>
								추가
							</button>
						</div>
						<!-- 목록 -->
						<div class="mb-2 max-h-96 flex-1 overflow-y-auto">
							{#if loading}
								<p class="py-4 text-center text-base-content/60">로딩 중...</p>
							{:else if people.length === 0}
								<p class="py-4 text-center text-base-content/60">인원이 없습니다</p>
							{:else}
								<div class="space-y-1">
									{#each [...people].sort((a, b) => b.skill - a.skill) as person, idx}
										{@const tier = getLolTier(person.skill)}
										<div class="flex items-center justify-between gap-2 rounded bg-base-200 p-2">
											<div class="flex items-center gap-2">
												<Icon icon={tier.icon} class="text-xl {tier.color}" />
												<span class="text-sm">{person.name}</span>
												{#if editingSkillId === person.id}
													<input
														type="number"
														min="0"
														max="100"
														class="input input-xs w-16"
														bind:value={tempSkill}
													/>
													<button
														class="btn btn-outline btn-xs btn-success"
														onclick={async () => {
															let skillValue = Number(tempSkill);
															if (isNaN(skillValue) || skillValue < 0) skillValue = 0;
															if (skillValue > 100) skillValue = 100;
															updateSkill(person, skillValue);
															editingSkillId = null;
														}}
													>
														저장
													</button>
												{:else}
													<span class="text-xs font-bold text-primary">{person.skill}</span>
													<button
														class="btn btn-outline btn-xs btn-info"
														onclick={() => {
															editingSkillId = person.id;
															tempSkill = person.skill;
														}}
													>
														수정
													</button>
												{/if}
											</div>
											<div class="flex gap-1">
												{#if editingSkillId !== person.id}
													<button
														class="btn btn-outline btn-xs btn-warning"
														onclick={() => addToWaitingList(person)}
														disabled={waitingList.some((p) => p.id === person.id)}
													>
														추가
													</button>
													<button
														class="btn btn-outline btn-xs btn-error"
														onclick={() => {
															if (confirm(`${person.name}을(를) 삭제하시겠습니까?`)) {
																removePerson(person);
															}
														}}
													>
														삭제
													</button>
												{/if}
											</div>
										</div>
									{/each}
								</div>
							{/if}
						</div>
						<!-- 액션 버튼 -->
						<div class="mt-auto">
							<button class="btn w-full btn-outline btn-sm" onclick={clearAll}> 초기화 </button>
						</div>
					</div>
				</div>
			</div>

			<div class="col-span-1 md:col-span-5">
				<!-- 티어 분포 카드 -->
				<div class="card mb-8 w-full bg-base-100 shadow-xl">
					<div class="card-body p-6">
						<h2 class="mb-4 card-title text-lg">🏆 티어 분포</h2>
						<div class="grid grid-cols-2 gap-2 sm:grid-cols-5 lg:grid-cols-10">
							{#each Object.entries(tierDistribution) as [tierName, tierInfo]}
								{@const tier = getLolTier(
									tierName === '아이언'
										? 5
										: tierName === '브론즈'
											? 15
											: tierName === '실버'
												? 25
												: tierName === '골드'
													? 35
													: tierName === '플래티넘'
														? 45
														: tierName === '에메랄드'
															? 55
															: tierName === '다이아'
																? 65
																: tierName === '마스터'
																	? 75
																	: tierName === '그랜드마스터'
																		? 85
																		: 95
								)}
								<div class="tooltip" data-tip="{tierName}: {tierInfo.range}점">
									<div
										class="cursor-pointer rounded-lg bg-base-200 p-3 text-center transition-colors hover:bg-base-300"
									>
										<Icon icon={tier.icon} class="text-2xl {tier.color} mx-auto mb-1" />
										<div class="text-xs font-semibold">
											{#if tierName === '그랜드마스터'}
												그랜드<br />마스터
											{:else}
												{tierName}
											{/if}
										</div>
										<div class="text-lg font-bold text-primary">{tierInfo.count}명</div>
									</div>
								</div>
							{/each}
						</div>
					</div>
				</div>

				<div class="grid w-full grid-cols-1 items-stretch justify-center gap-8 lg:grid-cols-3">
					<!-- 대기 리스트 -->
					<div
						class="card flex h-full flex-1 flex-col border-l-4 border-warning bg-base-100 shadow-xl"
					>
						<div class="card-body flex h-full flex-col p-6">
							<h2 class="card-title text-lg text-warning">
								⏳ 대기 리스트 ({waitingList.length}명)
							</h2>
							<!-- 대기 인원 목록 -->
							<div class="mb-4 max-h-[200px] flex-1 overflow-y-auto">
								{#if waitingList.length === 0}
									<p class="py-4 text-center text-base-content/60">대기 인원이 없습니다</p>
								{:else}
									<div class="space-y-1">
										{#each waitingList as person}
											{@const tier = getLolTier(person.skill)}
											<div class="flex items-center justify-between gap-2 rounded bg-base-200 p-2">
												<div class="flex items-center gap-2">
													<Icon icon={tier.icon} class="text-xl {tier.color}" />
													<span class="text-sm">{person.name}</span>
													<span class="text-xs font-bold text-warning">{person.skill}</span>
												</div>
												<button
													class="btn btn-outline btn-xs btn-error"
													onclick={() => removeFromWaitingList(person)}
												>
													✕
												</button>
											</div>
										{/each}
									</div>
								{/if}
							</div>
							<!-- 시뮬레이션 횟수 인풋 + 셔플 버튼 -->
							<div class="mt-auto flex items-center gap-2">
								<label class="flex items-center gap-1">
									<input
										type="number"
										min="1"
										max="100"
										class="input input-xs w-14 [appearance:textfield] [&::-webkit-inner-spin-button]:appearance-none [&::-webkit-outer-spin-button]:appearance-none"
										bind:value={simulationCount}
										style="text-align: right;"
									/>
								</label>
								<button
									class="btn flex-1 btn-warning"
									onclick={shuffleWaitingList}
									disabled={waitingList.length < 8}
								>
									셔플(8명↑)
								</button>
							</div>
						</div>
					</div>

					<!-- 팀원 선택 모달 -->
					{#if modalOpen}
						<div class="modal-open modal">
							<div class="modal-box max-w-md">
								<h3 class="mb-4 text-lg font-bold">
									{selectedTeam}팀 {selectedPosition + 1}번 선택
								</h3>
								<div class="max-h-80 space-y-2 overflow-y-auto">
									{#each availableOptions(selectedTeam, selectedPosition) as option}
										{@const tier = getLolTier(option.skill)}
										<button
											class="btn w-full justify-start btn-outline"
											onclick={() => selectPerson(option)}
										>
											<Icon icon={tier.icon} class="text-lg {tier.color}" />
											<span>{option.name}</span>
											<span class="text-xs font-bold">{option.skill}</span>
										</button>
									{/each}
									<!-- 빈 슬롯 옵션 -->
									<button
										class="btn w-full justify-start btn-outline"
										onclick={() => selectPerson(null)}
									>
										<span class="text-base-content/60">빈 슬롯</span>
									</button>
								</div>
								<div class="modal-action">
									<button class="btn" onclick={() => (modalOpen = false)}>취소</button>
								</div>
							</div>
						</div>
					{/if}

					<!-- A팀 -->
					<div
						class="card flex flex-1 flex-col border-l-4 border-primary bg-base-100 shadow"
						id="team-card-a"
					>
						<div class="card-body flex flex-col p-4">
							<h2 class="card-title text-lg text-primary">
								🔵 A팀 ({teamACount}/4)
								<span class="ml-2 text-xs text-primary">총점: {teamASum}점</span>
							</h2>
							<div class="flex-1 space-y-3">
								{#each Array(4) as _, idx}
									<div>
										<label class="text-xs text-base-content/70">{idx + 1}번</label>
										<button
											class="btn w-full justify-start btn-outline {teamAssignments.A[idx]
												? 'btn-primary'
												: ''}"
											onclick={() => openModal('A', idx)}
										>
											{#if teamAssignments.A[idx]}
												{@const tier = getLolTier(teamAssignments.A[idx].skill)}
												<Icon icon={tier.icon} class="text-lg {tier.color}" />
												<span>{teamAssignments.A[idx].name}</span>
												<span class="text-xs font-bold text-primary"
													>{teamAssignments.A[idx].skill}</span
												>
											{:else}
												<span class="text-base-content/60">선택하세요</span>
											{/if}
										</button>
									</div>
								{/each}
							</div>
						</div>
					</div>

					<!-- B팀 -->
					<div
						class="card flex flex-1 flex-col border-l-4 border-secondary bg-base-100 shadow"
						id="team-card-b"
					>
						<div class="card-body flex h-full flex-col p-4">
							<h2 class="card-title text-lg text-secondary">
								🟣 B팀 ({teamBCount}/4)
								<span class="ml-2 text-xs text-secondary">총점: {teamBSum}점</span>
							</h2>
							<div class="flex-1 space-y-3">
								{#each Array(4) as _, idx}
									<div>
										<label class="text-xs text-base-content/70">{idx + 1}번</label>
										<button
											class="btn w-full justify-start btn-outline {teamAssignments.B[idx]
												? 'btn-secondary'
												: ''}"
											onclick={() => openModal('B', idx)}
										>
											{#if teamAssignments.B[idx]}
												{@const tier = getLolTier(teamAssignments.B[idx].skill)}
												<Icon icon={tier.icon} class="text-lg {tier.color}" />
												<span>{teamAssignments.B[idx].name}</span>
												<span class="text-xs font-bold text-secondary"
													>{teamAssignments.B[idx].skill}</span
												>
											{:else}
												<span class="text-base-content/60">선택하세요</span>
											{/if}
										</button>
									</div>
								{/each}
							</div>
						</div>
					</div>
				</div>
			</div>
		</div>
	</div>
</div>
