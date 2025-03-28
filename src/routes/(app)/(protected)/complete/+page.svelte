<script lang="ts">
    import { completedCourses, parseCourseId } from '$lib/store.svelte';
    import { fly, slide } from 'svelte/transition';
    import { flip } from 'svelte/animate';
    import { onMount } from 'svelte';
    import CourseCard from '$lib/components/CourseCard.svelte';
    import Search from '$lib/components/Search.svelte';

    let loaded = $state(false);

    let creditsCompleted = $derived(Object.values(completedCourses.value).reduce((acc, course) => acc + (course.credit || 0), 0));
    
    let filterValue = $state('');

    let selectedDepartment = $state("All");

    let observeables = $derived.by(() => {
        if (selectedDepartment) {
            if (selectedDepartment === 'All') {
                return Object.entries(completedCourses.value);
            } else {
                return Object.entries(completedCourses.value).filter(([courseId, _]) => parseCourseId(courseId) === selectedDepartment);
            }
        } else {
            return Object.entries(completedCourses.value);
        }
    });
    
    let filteredCourses = $derived.by(() => {
        if (filterValue) {
            return observeables.filter(([_, courseInfo]) => {
                return courseInfo.course_name.toLowerCase().includes(filterValue.toLowerCase());
            });
        } else {
            return observeables;
        }
    })

    let departments = $derived(["All", ...Object.keys(completedCourses.value).reduce((acc, courseId) => {
        let parsedDepartment = parseCourseId(courseId);

        if (!acc.includes(parsedDepartment)) {
            acc.push(parsedDepartment);
        }

        return acc;

    }, Array<string>())]);

    let deptCreditAndCountMap = $derived(departments.reduce((acc, dept) => {
        // let deptCourses = Object.entries(allCourses.value).filter(([courseId, _]) => parseCourseId(courseId) === dept);
        // let deptCredits = deptCourses.reduce((acc, [_, course]) => acc + (course.credit || 0), 0);
        // return [dept, {count: deptCourses.length, credits: deptCredits}];

        if (dept === 'All') {
            //add all courses
            return new Map<string, {count: number, credits: number}> ([
                ['All', {count: Object.keys(completedCourses.value).length, credits: creditsCompleted}]
            ]);
        }

        let deptCourses = Object.entries(completedCourses.value).filter(([courseId, _]) => parseCourseId(courseId) === dept);
        let deptCredits = deptCourses.reduce((acc, [_, course]) => acc + (course.credit || 0), 0);
        

        if (!acc) {
            acc = new Map<string, {count: number, credits: number}>();
        }

        acc.set(dept, {count: deptCourses.length, credits: deptCredits});

        return acc;

    }, new Map<string, {count: number, credits: number}>()));

    onMount(() => {
        loaded = true;
    });

    function getAllCourseString(dept: string): string {
        if (dept === 'All') {
            return `(${deptCreditAndCountMap.get('All')?.count} courses, ${deptCreditAndCountMap.get('All')?.credits} Credits)`;
        } else {
            return `(${deptCreditAndCountMap.get(dept)?.count} courses, ${deptCreditAndCountMap.get(dept)?.credits} Credits)`;
        }
    }

</script>
{#if loaded}
<div class="filter">
    <!-- radio button -->
    {#each departments as dept, i}
        <div class="form-field" in:fly|global={{y: 5, delay: (i+1) * 50}}>
            <input type="radio" name="filter" id="{dept}" value="{dept}" bind:group={selectedDepartment} />
            <label for="{dept}" class="tag">
                <div class="dept">{dept}</div>
                {#if dept == selectedDepartment}
                <div class="tag-detail" transition:slide={{axis: "x"}}>{getAllCourseString(dept)}</div>
                {/if}
            </label>
        </div>
    {/each}
</div>
{#if observeables && observeables.length > 0}
<div class="container">
    <Search bind:filterValue={filterValue}/>
    <div class="courses">
        {#if filteredCourses.length == 0}
            <div class="empty">No courses found</div>
        {:else}
        {#each filteredCourses as [courseId, courseInfo], i (courseId)}
            <div class="course" animate:flip={{duration: 300}} in:fly|global={{y: 10, delay: 50 * (i+1)}}>
                <CourseCard courseId={courseId} cType="completed" grade={courseInfo.grade}/>
            </div>
        {/each}
        {/if}
    </div>
</div>
{:else}
    <div class="info no-course">No courses completed yet</div>
{/if}
{/if}

<style lang="scss">

    .container{
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: flex-start;
        height: 100%;
        width: 100%;
    }

</style>