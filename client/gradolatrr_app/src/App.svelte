<script>
// @ts-nocheck
    import { Router, Route } from "svelte-navigator";
    import { ApolloClient, InMemoryCache, HttpLink, ApolloLink } from "@apollo/client/core";
    import { setClient } from "svelte-apollo";
    import { createAuthLink } from "aws-appsync-auth-link";
    import { createSubscriptionHandshakeLink } from "aws-appsync-subscription-link";
    
    import Sidebar from "./sidebar/Sidebar.svelte";
    import Dashboard from "./dashboard/Dashboard.svelte";
    import Term from "./term/Term.svelte";
    import Course from "./course/Course.svelte";
    import CourseInfo from "./course/CourseInfo.svelte";
    import Assign from "./assign/Assign.svelte";
    import NewTerm from "./term/NewTerm.svelte";
    import NewCourse from "./course/NewCourse.svelte";
    import NewAssign from "./assign/NewAssign.svelte";
    import NewAssignBundle from "./assign/NewAssignBundle.svelte";
    import Settings from "./settings/Settings.svelte";
    import TooltipIcon from "./utils/TooltipIcon.svelte";

    let sidebarReload = false;
    let reload = false;
    let w = '15';
    let dragging = false;
    let body = document.body;

    const url = import.meta.env.VITE_APPSYNC_GRAPHQLENDPOINT;
    const region = import.meta.env.VITE_APPSYNC_REGION;
    const auth = {
        type: import.meta.env.VITE_APPSYNC_AUTHTYPE,
        apiKey: import.meta.env.VITE_APPSYNC_APIKEY
    };
    const httpLink = new HttpLink({ uri: url });
    const link = ApolloLink.from([
        createAuthLink({ url, region, auth }),
        createSubscriptionHandshakeLink({ url, region, auth }, httpLink),
    ]);
    const client = new ApolloClient({
        link,
        cache: new InMemoryCache(),
    });

    setClient(client);

    function triggerReload() {
        sidebarReload = !sidebarReload;
    }

    function collapseNav() {
        document.getElementById("navbar").style.display = "none";
        document.getElementById('navbar_show').style.display = 'block';
        document.getElementById("homepage").style.width = "100vw";
    }

    function showNav() {
        document.getElementById("navbar").style.display = "block";
        document.getElementById('navbar_show').style.display = 'none';
        document.getElementById("homepage").style.width = "83vw";
    }

    function clearJSEvents() {
        dragging = false;
        body.removeEventListener("mousemove", resize);
    }

    function resize(e) {
        if (e.pageX > 350 || e.pageX < window.innerWidth * 0.15) return;        
        w = (e.pageX / window.innerWidth) * 100 - 2;
    }

    function dividerMousedown(e) {
        e.preventDefault();
        dragging = true;
        body.addEventListener('mousemove', resize);
        body.style.setProperty("--left-width", e.pageX + 'px');
    }

    document.onmouseup = function() {
        dragging ? clearJSEvents() : '';
    };
    
</script>

<div>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.1.1/css/all.min.css" 
        integrity="sha512-KfkfwYDsLkIlwQp6LFnl8zNdLGxu9YAA1QvwINks4PhcElQSvqcyVLLD9aMhXd13uQjoXtEKNosOWaZqXgel0g==" 
        crossorigin="anonymous" referrerpolicy="no-referrer" />
</div>

<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<div>
  <Router>
      <div class="flex-row">
          <div id="navbar">
            <Sidebar class="sidebar" bind:w={w} bind:reload={sidebarReload} bind:triggerreload={reload} on:collapse={() => collapseNav()}/>
          </div>
          <div class="divider" on:mousedown={dividerMousedown} />
          <div id="homepage" style={`width: ${100-w}vw`} >
            <TooltipIcon icon='fa-solid fa-angles-right' id='navbar_show' click={showNav} position='right' text='expand'/>
        
            
            <Route path="/*">
                <Dashboard text="dashboard" bind:w={w} />
            </Route>
            <Route path="/new_course/:id/:name" let:params>
                <NewCourse term_id={params.id} term_name={params.name} 
                    on:message={triggerReload} />
            </Route>
            <Route path="/new_assign/:term_id/:term_name/:course_id/:course_name" let:params>
                <NewAssign term_id={params.term_id} term_name={params.term_name} 
                            course_id={params.course_id} course_name={params.course_name} />
            </Route>
            <Route path="/new_assignbundle/:term_id/:term_name/:course_id/:course_name" let:params>
                <NewAssignBundle term_id={params.term_id} term_name={params.term_name} 
                            course_id={params.course_id} course_name={params.course_name} />
            </Route>
            <Route path="/new_term">
                <NewTerm  on:message={triggerReload} />
            </Route>
            <Route path="/course/:term_id/:term_name/:id/:name" let:params>
                <Course id={params.id} name={params.name} term_id={params.term_id} 
                    term_name={params.term_name} bind:reload={reload} />
            </Route>
            <Route path="/term/:id/:name" let:params>
                <Term id={params.id} name={params.name}  on:message={triggerReload} />
            </Route>
            <Route path="/course/edit/:term_id/:term_name/:id/:name" let:params>
                <CourseInfo id={params.id} name={params.name} term_id={params.term_id} term_name={params.term_name}
                            on:message={triggerReload} bind:reload={reload} />
            </Route>
            <Route path="/assign/edit/:term_id/:term_name/:course_id/:course_name/:id/:name" let:params>
                <Assign id={params.id} name={params.name} course_id={params.course_id} course_name={params.course_name}
                        term_id={params.term_id} term_name={params.term_name} reload={reload} />
            </Route>
            <Route path="/settings" let:params>
                <Settings />
            </Route>
          </div>
      </div>
  </Router>
</div>  

<style>
#homepage {
  overflow-y: auto;
  overflow-x: hidden;
  height: 100vh;
}    

.divider {
    width: 3px;
    background-color: #F7F6F3;
}

.divider:hover {
    background-color: #e8e5df;
    cursor: col-resize;
}
</style>
