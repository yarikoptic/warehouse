<template>
<transition name="fade">
    <div v-if="open" class="brainlife-modal-overlay">
        <b-container class="brainlife-modal">
            <div class="brainlife-modal-header">
                <div class="brainlife-modal-header-buttons">
                    <div class="button" @click="open = false" style="margin-left: 20px; opacity: 0.8;">
                        <icon name="times" scale="1.5"/>
                    </div>
                </div>
                <span style="float: right; opacity: 0.3; margin-top: 10px;">
                    {{task.instance_id}} / {{task._id}}
                </span>
                <h4 style="margin-top: 5px;">
                    {{task.name}} 
                    <b style="opacity: 0.6; position: relative; top: -8px; font-size: 70%;">t.{{task.config._tid}}</b>
                </h4>
            </div><!--header-->

            <div class="scrolled" style="background-color: #fcfcfc;">
                <b-alert variant="secondary" :show="!loading && !smon.info" style="opacity: 0.8">Runtime info (_smon.out) not available</b-alert>

                <div class="block">
                    <b-row>
                        <b-col cols="6">
                            <h5>Task</h5>
                            <!--dates table-->
                            <table class="table table-sm">
                            <tr>
                                <th>Submitter</th>
                                <td>
                                    <contact :id="task.user_id" size="small"/>
                                </td>
                            </tr>
                            <tr>
                                <th>Created</th>
                                <td>{{new Date(task.create_date).toLocaleString()}}</td>
                            </tr>
                            <tr v-if="task.start_date">
                                <th>Started</th>
                                <td>
                                    {{new Date(task.start_date).toLocaleString()}}
                                    (<timeago :datetime="task.start_date" :auto-update="1"/>)
                                </td>
                            </tr>
                            <tr v-if="task.finish_date">
                                <th>Finished</th>
                                <td>
                                    {{new Date(task.finish_date).toLocaleString()}}
                                </td>
                            </tr>
                            <tr v-if="task.finish_date">
                                <th>Runtime</th>
                                <td>
                                    {{getruntime}}
                                </td>   
                            </tr>
                            <tr v-if="task.fail_date">
                                <th>Failed</th>
                                <td>{{new Date(task.fail_date).toLocaleString()}}</td>
                            </tr>
                            <tr v-if="task.nice">
                                <th>Nice</th>
                                <td>{{task.nice}} <small style="opacity: 0.5">yeilds to less nice tasks</small></td>
                            </tr>
                            <tr v-if="task.config._rule">
                                <th>Rule</th>
                                <td>
                                    <small>This task was submitted by {{task.config._rule.id}}
                                    For subject:<b>{{task.config._rule.subject}}</b></small>
                                    <span v-if="task.config._rule.session">session:<b>{{task.config._rule.session}}</b></small></span>
                                </td>
                            </tr>
                            <!-- this confuses people with job walltime
                            <tr v-if="task.max_runtime">
                                <th>Max Runtime</th>
                                <td>{{task.max_runtime/(1000*60)}} mins</td>
                            </tr>
                            -->
                            <tr v-if="resource">
                                <th>Resource</th>
                                <td>{{resource.name}} <small>{{resource.config.username}}@{{resource.config.hostname}} {{resource.desc}}</small></td>
                            </tr>
                            <tr v-if="task.next_date" style="opacity: 0.3;">
                                <th>Next&nbsp;Chk</th>
                                <td>in {{((new Date(this.task.next_date).getTime() - new Date().getTime())/(1000*60)).toFixed(0)}} mins</td>
                            </tr>
                            </table>
                        </b-col>
                        <b-col cols="6" v-if="smon.info">
                            <h5>Compute Node</h5>
                            <!--<p><small style="opacity: 0.5">This task rans on the following compute node</small></p>-->
                            <table class="table table-sm">
                            <tr>
                                <th>Host</th>
                                <td>
                                    {{smon.info.uid}}@{{smon.info.uname[1]}}
                                </td>
                            </tr>
                            <tr>
                                <th>Session ID</th>
                                <td>
                                    {{smon.info.sid}}
                                </td>
                            </tr>
                            <tr>
                                <th>OS</th>
                                <td>
                                    {{smon.info.uname[3]}}<br>
                                    <span style="opacity: 0.7">{{smon.info.uname[0]}} {{smon.info.uname[2]}}</span>
                                </td>
                            </tr>
                            <tr>
                                <th>CPUs</th>
                                <td>
                                    {{smon.info.cpu_total}} Total
                                    <span style="opacity: 0.7" v-if="smon.info.cpu_total != smon.info.cpu_avail">({{smon.info.cpu_requested}} requested)</span>
                                </td>
                            </tr>
                            <tr>
                                <th>Memory</th>
                                <td>
                                    {{smon.info.memory_total|memory}} Total
                                    <span style="opacity: 0.7" v-if="smon.info.memory_total != smon.info.memory_avail">({{smon.info.memory_requested|memory}} requested)</span>
                                </td>
                            </tr>
                            <tr v-if="smon.info.walltime_requested">
                                <th>Requested Walltime</th>
                                <td v-if="smon.info.walltime_requested < 3600*60">
                                    {{smon.info.walltime_requested/60}} mins
                                </td>
                                <td v-else>
                                    {{smon.info.walltime_requested/3600}} hours
                                </td>
                            </tr>
                            <tr v-if="smon.info.gpus">
                                <th>GPUs</th>
                                <td>
                                    <p v-for="gpu in smon.info.gpus" :key="gpu.index">
                                        <small title="bus_id">{{gpu.bus_id}}</small><br>
                                        {{gpu.name}} | driver: {{gpu.driver_version}}<br>
                                        Memory: {{gpu['memory.total']}} | Mode: {{gpu.compute_mode}}<br>
                                    </p>
                                </td>
                            </tr>
                            </table>
                        </b-col>
                    </b-row>
                </div>

                <h5 v-if="loading" class="block" style="opacity: 0.8;"><icon name="cog" :spin="true"/> Loading runtime info .. </h5>

                <div v-if="!loading && smon.info" class="block">

                    <h5>Resource Utilization</h5>
                    <p style="opacity: 0.7"><small>
                        The following information is collected on the actual compute node used to run this App.
                        The App should be using close to all requested CPU cores / memory.
                    </small></p>
               
                    <!-- without watchShallow, vue-plotly will go infinite loop with layout watch event and locks up the browser-->
                    <div v-if="smon.cpu.data">
                        <ExportablePlotly :data="smon.cpu.data" :layout="smon.cpu.layout" :watchShallow="true"/>
                    </div>
                    <div v-if="smon.memory_rss.data">
                        <ExportablePlotly :data="smon.memory_rss.data" :layout="smon.memory_rss.layout" :watchShallow="true"/>
                    </div>
                    <div v-if="smon.memory_vsz.data">
                        <ExportablePlotly :data="smon.memory_vsz.data" :layout="smon.memory_vsz.layout" :watchShallow="true"/>
                    </div>
                    <div v-if="smon.disk.data">
                        <ExportablePlotly :data="smon.disk.data" :layout="smon.disk.layout" :watchShallow="true"/>
                    </div>
                    <br>

                    <h5>Job ENVs</h5>
                    <p style="opacity: 0.8; margin: 5px 0px; overflow: auto;">
                        <b-badge v-for="(v, k) in smon.info.env" :key="k" variant="light"><b>{{k}}</b> {{v}}</b-badge>
                    </p>
                </div> <!-- smon-->
                <div class="block">
                    <h5>Status Changes</h5>
                    <taskevents :taskId="task._id"/>
                </div>
                <div class="block">
                    <h5>Config</h5>
                    <!--<pre v-highlightjs="JSON.stringify(task.config, null, 4)" style="background-color: white;"><code class="json hljs"></code></pre>-->
                    <editor ref="task-config" @init="editorInit" v-model="taskConfigJson" lang="json"/>
                </div>
            </div>
        </b-container>
    </div>
</transition>
</template>

<script>
import Vue from 'vue'

const lib = require('@/lib');

export default {
    components: { 
        app: ()=>import('@/components/app'), 
        datatypetag: ()=>import('@/components/datatypetag'), 
        configform: ()=>import('@/components/configform'), 
        tageditor: ()=>import('@/components/tageditor'), 
        contact: ()=>import('@/components/contact'), 
        ExportablePlotly: ()=>import('@/components/ExportablePlotly'),
        taskevents: ()=>import('@/components/taskevents'),

        VueMarkdown: ()=>import('vue-markdown'), 

        editor: ()=>import('vue2-ace-editor'), 
    },

    data() {
        return {
            open: false,
            task: null,
            resource: null,
            smon: {
                info: null,

                //plotly graphs
                disk: {
                    data: null,
                    layout: null,
                },

                cpu: {
                    data: null,
                    layout: null,
                },

                memory_rss: {
                    data: null,
                    layout: null,
                },

                memory_vsz: {
                    data: null,
                    layout: null,
                },
            },
            taskConfigJson: "",
            loading: false,
        }
    },

    created() {
        this.$root.$on("taskinfo.open", opt=>{
            this.task = opt.task;
            this.resource = opt.resource;
            this.open = true;
            this.loadSmon();
            this.taskConfigJson = JSON.stringify(this.task.config, null, 4);
        });

        //TODO - call removeEventListener in destroy()? Or I should do this everytime modal is shown/hidden?
        document.addEventListener("keydown", e => {
            if (e.keyCode == 27) {
                this.open = false;
            }
        });
    },

    methods: {
        editorInit(editor) {
            lib.editorInit(editor, error => {
                editor.container.style.lineHeight = 1.25;
                editor.renderer.updateFontSize();
                editor.setReadOnly(true);  // false to make it editable

                editor.setAutoScrollEditorIntoView(true);
                editor.setOption("maxLines", 30);
                editor.setOption("minLines", 3);

                editor.setShowPrintMargin(true);
            });
        },

        loadSmon() {
            this.loading = true;
            this.smon.info = null;
            this.$http.get(Vue.config.amaretti_api+'/task/download/'+this.task._id+'/_smon.out', {
                //if _smon.out only contains a single entry, axios will parse it as json.. which fails with .split("\n") below.
                //let's prevent any transformation.
                transformResponse: (res) => {
                    return res;
                },
            }).then(res=>{
                let records = res.data.split("\n");

                //first record should be host/job info
                this.smon.info = JSON.parse(records.shift()); 
                let begin_date = new Date(this.smon.info.time*1000);
                let end_date =  new Date(); //default to now.. in case we don't have any data

                function shorten(long) {
                    if(long.length < 50) return long;
                    let tokens = long.split(" ");
                    let short = "";
                    tokens.forEach(t=>{
                        if(t.length < 20) {
                            short+=t+" ";
                            return;
                        }
                        short+=".."+t.substring(t.length-20)+" ";
                    });
                    if(short.length > 50) short = short.substring(0, 50)+"...";
                    return short;
                }

                //rest is resource utilization 
                let disk = {
                    x: [],
                    y: [],
                    name: "workdir (.)",
                    mode: "lines",
                    line: {width: 0},
                    fill: 'tozeroy'
                }; 

                let pcpus = {};

                let memory_rss = {};
                let memory_vsz = {};

                let gpus_gpu = {};
                let gpus_memory = {};
                let gpus_temp = {};

                //first organize data into different groups
                records.forEach(record_json=>{
                    if(record_json == "") return;
                    try {
                        let record = JSON.parse(record_json);
                        let time = new Date(record.time*1000);
                        end_date = time;

                        //deal with disk usage
                        if(record.disks) record.disks.forEach(rd=>{
                            if(rd.path == ".") {
                                disk.x.push(time);
                                disk.y.push(rd.size/1000);
                            }
                        });

                        //deal with process
                        if(record.processes) record.processes.forEach(p=>{
                            if(p.cmd.indexOf("ps ") == 0) return;
                            if(!pcpus[p.pid]) pcpus[p.pid] = {x: [], y: [], name: shorten(p.cmd), mode: 'lines', stackgroup: 'pcpu', line: {width: 0} }
                            pcpus[p.pid].x.push(time);
                            pcpus[p.pid].y.push(p.pcpu);

                            if(!memory_rss[p.pid]) memory_rss[p.pid] = {x: [], y: [], name: shorten(p.cmd), mode: 'lines', stackgroup: 'memory', line: {width: 0} }
                            memory_rss[p.pid].x.push(time);
                            memory_rss[p.pid].y.push((p.rss*1024)/(1000*1000*1000)); //convert kiloBytes to GB

                            if(!memory_vsz[p.pid]) memory_vsz[p.pid] = {x: [], y: [], name: shorten(p.cmd), mode: 'lines', stackgroup: 'memory', line: {width: 0} }
                            memory_vsz[p.pid].x.push(time);
                            memory_vsz[p.pid].y.push((p.vsz*1024)/(1000*1000*1000)); //convert kiloBytes to GB
                        });

                        //deal with gpu info
                        if(record.gpus) record.gpus.forEach(g=>{
                            if(!gpus_gpu[g.index]) {
                                gpus_gpu[g.index] = {x: [], y: [], name: g.name, mode: 'lines', yaxis: "y2"}
                                gpus_memory[g.index] = {x: [], y: [], name: g.name, mode: 'lines', yaxis: "y2"}
                                gpus_temp[g.index] = {x: [], y: [], name: g.name+" temperature"}
                            }
                            gpus_gpu[g.index].x.push(time);
                            gpus_memory[g.index].x.push(time);
                            gpus_temp[g.index].x.push(time);

                            gpus_gpu[g.index].y.push(g['utilization.gpu']);
                            gpus_memory[g.index].y.push(g['utilization.memory']);
                            gpus_temp[g.index].y.push(g['temperature.gpu']);
                        });
                    } catch(err) {
                        //parse error?
                        console.log(record_json);
                        console.error(err);
                    }
                });

                //then construct plotly data/layout
                this.smon.disk.data = [disk];
                this.smon.disk.layout = {
                    title: "Disk Usage",
                    height: 370,
                    margin: {
                        //l: 50, 
                        //r: 10, 
                        //b: 80,
                        t: 25,
                    },
                    showlegend: true,
                    legend: {
                        //orientation: "h",
                        font: {
                            family: 'sans-serif',
                            size: 10,
                        }
                    },
                    xaxis: {
                        range: [begin_date, end_date],
                    },
                    yaxis: {
                        title: 'MB',
                        //range: [new Date("2018-01-01"), new Date()],
                    },
                };

                this.smon.cpu.data = [];
                for(let pid in pcpus) {
                    //ignore process with all near-0 memory usage
                    let non0 = pcpus[pid].y.find(y=>y>0.5);
                    if(non0) this.smon.cpu.data.push(pcpus[pid]);
                }
                this.smon.cpu.layout = {
                    title: "CPU Usage (ps/pcpu)",
                    xaxis: {
                        range: [begin_date, end_date],
                    },
                    yaxis: {
                        title: 'CPU%',
                        //overlaying: "y",
                        //side: "right",
                        //type: 'log',
                    },
                    showlegend: true,
                    legend: {
                        font: {
                            family: 'sans-serif',
                            size: 10,
                        },
                        x: 1.1,
                    },
                    height: 400,
                    margin: {
                        t: 25,
                    },
                    shapes: [
                        //job max
                        {
                            type: 'rect',
                            xref: 'paper',
                            x0: 0,
                            y0: 100.0*this.smon.info.cpu_requested*0.95,
                            x1: 1,
                            y1: 100.0*this.smon.info.cpu_requested,
                            fillcolor: 'rgba(255, 0, 0, 0.5)',
                            line: {
                                width: 0,
                            }
                        },
                    ]
                };

                this.smon.memory_rss.data = [];
                for(let pid in memory_rss) {
                    //ignore process with all near-0 memory usage
                    let non0 = memory_rss[pid].y.find(y=>y>0.01); //10MB min 
                    if(non0) this.smon.memory_rss.data.push(memory_rss[pid]);
                }
                this.smon.memory_vsz.data = [];
                for(let pid in memory_vsz) {
                    //ignore process with all near-0 memory usage
                    let non0 = memory_vsz[pid].y.find(y=>y>0.01); //10MB min
                    if(non0) this.smon.memory_vsz.data.push(memory_vsz[pid]);
                }
                this.smon.memory_rss.layout = {
                    title: "Memory Usage (RSS)",
                    xaxis: {
                        range: [begin_date, end_date],
                    },
                    yaxis: {
                        title: 'GB',
                        //overlaying: "y",
                        //side: "right",
                        //type: 'log',
                    },
                    showlegend: true,
                    //paper_bgcolor: "#000",
                    legend: {
                        font: {
                          family: 'sans-serif',
                          size: 10,
                        },
                        x: 1.1,
                    },
                    height: 400,
                    margin: {
                        t: 25,
                    },
                    shapes: [
                        //job memory max
                        {
                            type: 'rect',
                            //xref: 'paper',
                            x0: begin_date,
                            y0: this.smon.info.memory_requested/(1000*1000*1000)*0.95,
                            x1: end_date,
                            y1: this.smon.info.memory_requested/(1000*1000*1000),
                            fillcolor: 'rgba(255, 0, 0, 0.5)',
                            line: {
                                width: 0,
                            }
                        },

                    ]
                };
                this.smon.memory_vsz.layout = Object.assign({}, this.smon.memory_rss.layout, {
                    title: "Virtual Memory Usage (VSZ)",
                });

                if(this.smon.info.gpus) {
                    for(let id in gpus_gpu) {
                        this.smon.cpu.data.unshift(gpus_gpu[id]);
                        this.smon.memory_rss.data.unshift(gpus_memory[id]);
                        //this.smon.gpu.data.push(gpus_temp[id]);
                    }

                    //add another axis
                    Object.assign(this.smon.cpu.layout, {
                        title: "CPU (ps/pcpu) & GPU (nvidia-smi) Usage",
                        yaxis2: {
                            title: 'GPU%',
                            overlaying: 'y',
                            side: 'right',
                            //position: 0.90,
                            range: [0, 105],
                        },
                    });
                    //add another axis
                    Object.assign(this.smon.memory_rss.layout, {
                        yaxis2: {
                            title: 'GPU%',
                            //anchor: 'x',
                            overlaying: 'y',
                            side: 'right',
                            //position: 0.90,
                            range: [0, 105],
                        },
                    });
                }

                this.loading = false;
            }).catch(err=>{
                this.loading = false;
                console.error(err);
            });
        },
    },

    computed: {

        getruntime() {
            let diff = new Date(new Date(this.task.finish_date) - new Date(this.task.start_date));
            return diff.toISOString().substr(11, 8);
       }
    },

    filters: {
        memory(v) {
            return (v/(1000*1000*1000)).toFixed(2)+"GB";
        }
    },
} 
</script>

<style scoped>
h5 {
    font-size: 17px;
    font-weight: bold;
    color: gray;
    text-transform: uppercase;
}
.scrolled {
    position: absolute;
    top: 60px;
    left: 0;
    right: 0;
    bottom: 0px;
    overflow-x: hidden;
    overflow-y: auto;
}
.badge {
    border-radius: 0;
    margin-right: 3px;
    font-weight: normal;
    padding: 5px;
}
.block {
    padding: 20px;
}
</style>
