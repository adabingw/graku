<script>
// @ts-nocheck

    import { query, mutation } from 'svelte-apollo';
    import { navigate } from 'svelte-navigator';
    import { createEventDispatcher, onDestroy } from 'svelte';

    import Folder from '../utils/Folder.svelte';
    import InfoTable from '../utils/InfoTable.svelte';
    import HeaderField from '../utils/HeaderField.svelte';
    import { COURSE_INFO } from "../constants/queries_get";
    import { DELETE_COURSE, DELETE_ASSIGN } from '../constants/queries_delete';
    import { UPDATE_COURSE, UPDATE_ASSIGNMENT } from '../constants/queries_put';
    import TooltipIcon from '../utils/TooltipIcon.svelte';
    
    export let id;
    export let name;
    export let term_id; 
    export let term_name;
    export let reload;
    let del = false;

    const dispatch = createEventDispatcher();

    let query_result = query(COURSE_INFO, {
        variables: { id }
    });
    let info;
    let last_info;
    let name_change = name;
    let delete_course = mutation(DELETE_COURSE);
    let delete_assign = mutation(DELETE_ASSIGN);
    let update_course = mutation(UPDATE_COURSE);
    let update_assign = mutation(UPDATE_ASSIGNMENT);

    onDestroy(() => {
        console.log("onDestroy for Course")
        saveChanges();
    })

    async function saveChanges() {
        if (del) return;

        if (!name) {
            alert("Name cannot be empty!");
            return;
        }
        
        try {
            let content_info = JSON.parse(info["getCourse"]["content_info"]);
            for (let key of Object.keys(content_info)) {
                if (content_info[key]["old_type"] != undefined) {
                    let old_type = content_info[key]["old_type"];
                    let new_type = content_info[key]["type"];
                    let assignments = info["getCourse"]["assignments"];

                    for (let i = 0; i < assignments.length; i++) {
                        let assign = assignments[i];
                        let assign_data = JSON.parse(assign["data"]);
                        assign_data[key]["type"] = new_type;
                        if (old_type == "number" || old_type == "text" || old_type == "checked") {
                            if (new_type == "multiselect" || new_type == "singleselect") {
                                assign_data[key]["content"] = [];
                                assign_data[key]["tag_info"] = content_info[key]["tag_info"];
                            } else if (new_type == "date") {
                                assign_data[key]["content"] = (new Date()).toISOString().split('T')[0];
                            } 
                        } else if (old_type == "multiselect" || old_type == "singleselect") {
                            if (new_type == "singleselect" || new_type == "multiselect") {
                                assign_data[key]["content"] = [];
                            } else {
                                if (new_type == "text") {
                                    assign_data[key]["content"] = "";
                                } else if (new_type == "date") {
                                    assign_data[key]["content"] = (new Date()).toISOString().split('T')[0];
                                } else if (new_type == "number") {
                                    assign_data[key]["content"] = 0;
                                } else if (new_type == "checked") {
                                    assign_data[key]["content"] = false;
                                }
                                delete assign_data[key]["tag_info"];
                            }
                        }

                        await update_assign({
                            variables: {
                                input: {
                                    id: assign["id"], 
                                    type: "item", 
                                    data: JSON.stringify(assign_data)
                                }
                            }
                        })
                    }
                    delete content_info[key]["old_type"]
                    info["getCourse"]["content_info"] = JSON.stringify(content_info);
                }
            }

            await update_course({
                variables: {
                    input: {
                        id: id, 
                        type: "course",
                        term_id: term_id, 
                        name: name_change, 
                        data: info["getCourse"]["data"],
                        content_info: info["getCourse"]["content_info"]
                    }
                }
            });

            query_result.refetch({ id });

            if (name_change != name) {
                dispatch('message', {
                    text: "reload"
                });
                name_change = name;
            }

            reload = !reload;
        } catch(error) {
            console.error(error);
        }
    }

    async function deleteCourse() {
        let confirmDelete = confirm("Delete this course?");
        if (!confirmDelete) return;

        del = true;

        try {
            await delete_course({ 
                variables: { 
                    input: {
                        id: id, 
                        type: "course"
                    }
                } 
            });

            for (let i = 0; i < info["getCourse"]["assignments"].length; i++) {
                let assign_id = info["getCourse"]["assignments"][i]["id"];
                await delete_assign({
                    variables: {
                        input: {
                            id: assign_id, 
                            type: "item"
                        }
                    }
                });
            }
        } catch (error) {
            console.error(error);
        }
        dispatch('message', {
            text: "reload"
        });
        navigate("/");
    }

    function updateChange(event) {
        info["getCourse"]["data"] = event.detail.data;
    }

    async function newData(event) {
        let action = event.detail.action;
        let assignments = info["getCourse"]["assignments"];

        for (let i = 0; i < assignments.length; i++) {
            let assign = assignments[i];
            let assign_data = JSON.parse(assign["data"]);

            if (action == 'saved') {
                let name = event.detail.info_name;
                let info = event.detail.new_info;
                let new_property = {
                    "type": info.type, 
                }

                if (info.type == "number") new_property["content"] = 0;
                else if (info.type == "text") new_property["content"] = "";
                else if (info.type == "multiselect" || info.type == "singleselect") {
                    new_property["content"] = [];
                    new_property["tag_info"] = [];
                } else if (info.type == "date") {
                    new_property["content"] = (new Date()).toISOString().split('T')[0]
                } else if (info.type == "checked") {
                    new_property["content"] = false;
                }

                assign_data[name] = new_property;
            } else if (action == 'delete') {
                let name = event.detail.info_name;
                delete assign_data[name]
            } else if (action == 'update') {
                let old_name = event.detail.old_name;
                let new_name = event.detail.new_name;
                assign_data[new_name] = assign_data[old_name];
                delete assign_data[old_name];
            } 

            await update_assign({
                variables: {
                    input: {
                        id: assign["id"], 
                        type: "item", 
                        data: JSON.stringify(assign_data)
                    }
                }
            })
        }
    }

    $: {
        if ($query_result.data != undefined && (JSON.stringify(last_info) == JSON.stringify(info))) {
            info = JSON.parse(JSON.stringify(Object.assign({}, $query_result.data)));
            last_info = JSON.parse(JSON.stringify(info));
        }
    }

    $: {
        id;
        query_result.refetch({ id });
        last_info = info;
    }

    $: {
        info;
        last_info = undefined;
    }

</script>

<div class="page">
    <Folder term_id={term_id} term_name={term_name} course_id={id} course_name={name} assign_name={""} />
    <HeaderField bind:inputText={name} text="" on:message={(event) => {name_change = event.detail.data;}}/>
    {#if info != undefined}
        <InfoTable cmd="course" bind:info={info.getCourse} on:message={updateChange} on:action={newData} />
    {/if}
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <div class="term-op">
        <TooltipIcon icon='fa-solid fa-trash-can' position='left' text='delete' click={deleteCourse} />
        <TooltipIcon icon='fa-solid fa-floppy-disk' click={saveChanges} position='left' text='save'/>
    </div>
</div>

<style>

</style>
