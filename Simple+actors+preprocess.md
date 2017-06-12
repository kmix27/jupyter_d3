

```python
import pandas as pd 
import numpy as np 
%matplotlib inline
pd.set_option('display.max_columns',75)
pd.set_option('display.max_rows',150)
sns.set_style('darkgrid')
import networkx as nx
from itertools import combinations, chain, count
from networkx.readwrite import json_graph
from networkx.utils import make_str
import json
```


```python
# tweak our display
from IPython.display import HTML
HTML('''<style>.CodeMirror{min-width:100% !important;}div#notebook-container    { width: 90%; }</style>''')

```




<style>.CodeMirror{min-width:100% !important;}div#notebook-container    { width: 90%; }</style>



### lets do some viz in jupyter!
we'll be using this form for our viz: https://bl.ocks.org/mbostock/4062045  

This is a walkthrough of the steps I took in pandas, and NetworkX to convert my csv's first into a graph structure,  and finally into a d3 viz.  Ultimately I would like this viz to run inline in my notebook, this is certainly possible, but because it's pretty hacky it's not working at the moment.  I have successfully done this for simpler visualizations, so it is possible.    

There was a **SIGNIFICANT** amount of webscraping, html parsing, and data cleaning that came before,  but it took a while for me to wrap my head around graph structures, so hopefully this helps. 


```python
# global variable for the CSV file locations
DATAPATH = '/Users/kylemix/ds/metis/metisgh/luther'
```


```python
# load in our csv data
df = pd.read_csv('data/actors_stacked_G.csv')
fin = pd.read_csv('data/dom_bo_fin_y.csv')
```


```python
# merge the two tables such that we have an edge weight of domestic box office ROI for every actor on the project
matrix_stacked = pd.merge(df[['title','name']], fin[['title','domestic_box_office_roi']],'left',left_on='title',right_on='title' )
# remove NaN values
matrix_stacked =  matrix_stacked.dropna()
```

Our dataset is pretty substantial, there are over 4k nodes and upwards of 1.5million edges.  This will not look very good if we pass it directly to D3, and as the json file alone is something like 130mb, you'll porbably crash your browser.  SO we need to filter.  we'll look for now at actors that have at least 25 projects under their belt so as to reduce our node and edge count to something that is at least moderatly visualizable


```python
# create and filter the table used for building up the graph 
# get a project count accross actors
# the first datapoint is a fluke I think and thus I'm omitting it
M = matrix_stacked.name.value_counts().reset_index().iloc[1:]
# correct the column labels
M.columns = ['name', 'counts']
# filter such that only actors that have more that n projects are included setting n to 25 for now
M = M[M.counts >25]
```


```python
# build a collaboration list of 3 item tuples (i[0] = node 1, i[1] = node 2, i[2] = dom_ROI weight)
#this is the point we could add more datapoints, and for an interesting viz you probably should.
temp = matrix_stacked.set_index('title')
fi = fin.set_index('title')
colaboration_list =  []
for movie in matrix_stacked.title.unique():
#     filter financials table for roi val
    r = fi.loc[movie].domestic_box_office_roi
#     build list of actors that have n or more projects from the M table above
    actors = [act for act in list(temp.loc[movie].name) if act in list(M.name)]
#     get all possible pairs
    tc = combinations(actors,2)
#     instantiate temp list to hold out tuples
    c = []
#     iterate over combinations and save the edge with it's weight
    for pair in tc:
        c.append((pair[0],pair[1],r))
#    Finally extend our collaboration list with the new wighted collaborations
    colaboration_list.extend(c)    
```

Now we'll use NetworkX to build our graph datastructure


```python
# build a weighted graph from a collaboration list of tuples
# of note,  NetworkX will automatically add nodes if you ask it to creat an edge to a non-existent node,  so we can skip the add nodes loop
g = nx.Graph()
for i in colaboration_list:
#     checks for the existence of an edge, if exists it increments the colab count and adds the roi weight to the existing total
    if g.has_edge(i[0], i[1]):
        g[i[0]][i[1]]['weight'] += i[2]
        g[i[0]][i[1]]['count'] +=1
# if not exists, adds the edge and sets the count at 1, and the weight to the weight in the tuple
    else:
        g.add_edge(i[0],i[1],weight=i[2],count=1)
```

Normally we would simply call  
``` python 
nx.node_link_data(g) 
``` 
and it would do the work to put it in the format we need.  Unfortunately networkx forces the source and target values for link data to be integers, to fix that below is an altered version of the networkx function "node_link_data"  it returns a D3 friendly json graph that will send strings to the source and target valuse of the edge data.  The original function forces an integer ID and thus nothing links when you import into D3.  

see : https://stackoverflow.com/questions/38757701/targets-dont-match-node-ids-in-networkx-json-file


```python


_attrs = dict(id='id', source='source', target='target', key='key')

def node_link_data_fixed(G, attrs=_attrs):
    """Return data in node-link format that is suitable for JSON serialization
    and use in Javascript documents.
    """
    multigraph = G.is_multigraph()
    id_ = attrs['id']
    source = attrs['source']
    target = attrs['target']
    # Allow 'key' to be omitted from attrs if the graph is not a multigraph.
    key = None if not multigraph else attrs['key']
    if len(set([source, target, key])) < 3:
        raise nx.NetworkXError('Attribute names are not unique.')
    mapping = dict(zip(G, count()))
    data = {}
    data['nodes'] = [dict(chain(G.node[n].items(), [(id_, n)])) for n in G]
    if multigraph:
        data['links'] = [
            dict(chain(d.items(),
                       [(source, u), (target,v), (key, k)]))
            for u, v, k, d in G.edges_iter(keys=True, data=True)]
    else:
        data['links'] = [
            dict(chain(d.items(),
                       [(source, u), (target, v)]))
            for u, v, d in G.edges_iter(data=True)]
    return data
```

Now we have a solid function  we can proceed to  build up a dict we can serialize in json.  Because we fixed the NetworkX function it is in the format we need to pass to D3 for a graph viz.  The ouput of this function is a dict with keys ['nodes', 'links'].  Nodes is a list of dicts where key= 'id' = str name of actor.  Links is also a list of dicts with data keys of 'count' and 'weight' where weight is the domestic ROI of collaboration, and 'source' = str actor name 1  and 'target' = str actor name 2. Now that we have this it's pretty straightforward to plug into D3, Ideally we'll do this inline here, but we can also save to json and load the json normally in an external html file.


```python
graph = node_link_data_fixed(g)
```

Now we can serialize our newly created data into json file in the correct dir where our project file lives


```python
 json.dump(d, open('viz/force3.json','w'))
```

## OR add inline d3 code here
### (this is still buggy,  working on it)


```python
# make our data visible to the window 
from IPython.display import Javascript
Javascript("""
           window.graph={};
           """.format(graph))
```




    <IPython.core.display.Javascript object>




```python
// set up our access to d3 source
%%javascript
require.config({
    paths: {
        d3: '//cdnjs.cloudflare.com/ajax/libs/d3/4.9.1/d3.js'
    }
});
```


    <IPython.core.display.Javascript object>



```python
%%javascript
require(['d3'], function(d3){
  //a weird idempotency thing
  $("#chart1").remove();
  //create canvas
  element.append("<div id='chart1'></div>");
  $("#chart1").width("800px");
  $("#chart1").height("800px");

var svg = d3.select("#chart1").append("svg")
    .style("position", "relative")
    .style("background-color","white")
    .attr('width',800)
    .attr('height',800)
// build our viz    
var graph = window.graph    
var width = +svg.attr("width"),
    height = +svg.attr("height");

var color = d3.scaleOrdinal(d3.schemeCategory20);

var simulation = d3.forceSimulation()
    .force("link", d3.forceLink().id(function(d) {return d.id}))
    .force("charge", d3.forceManyBody().strength(-900))
    .force("center", d3.forceCenter(width / 2, height / 2));



  var link = svg.append("g")
      .attr("class", "links")
    .selectAll("line")
    .data(graph.links)
    .enter().append("line")
      .attr("stroke-width", function(d) { return Math.sqrt(d.weight); });

  var node = svg.append("g")
      .attr("class", "nodes")
    .selectAll("circle")
    .data(graph.nodes)
    .enter().append("circle")
      .attr("r", 5)
      .attr("fill","steelblue")
      // .attr("fill", function(d) { return color(d.group); })
      .call(d3.drag()
          .on("start", dragstarted)
          .on("drag", dragged)
          .on("end", dragended));

  node.append("title")
      .text(function(d) { return d.id; });

  simulation
      .nodes(graph.nodes)
      .on("tick", ticked);

  simulation.force("link")
      .links(graph.links);

  function ticked() {
    link
        .attr("x1", function(d) { return d.source.x; })
        .attr("y1", function(d) { return d.source.y; })
        .attr("x2", function(d) { return d.target.x; })
        .attr("y2", function(d) { return d.target.y; })
        ;

    node
        .attr("cx", function(d) { return d.x; })
        .attr("cy", function(d) { return d.y; });
  }


function dragstarted(d) {
  if (!d3.event.active) simulation.alphaTarget(0.3).restart();
  d.fx = d.x;
  d.fy = d.y;
}

function dragged(d) {
  d.fx = d3.event.x;
  d.fy = d3.event.y;
}

function dragended(d) {
  if (!d3.event.active) simulation.alphaTarget(0);
  d.fx = null;
  d.fy = null;
}
});


```


    <IPython.core.display.Javascript object>



```python

```


```python

```
