# Supply Chain Logisitics Problem
This problem is shared by Brunel University London in this [Link](https://brunel.figshare.com/articles/dataset/Supply_Chain_Logistics_Problem_Dataset/7558679)

## What Question we are trying to solve in this problem?
How do we plan our daily orders fulfilment?

### Further Explanation: 
The question "how" suggests that this is a prescriptive analysis level problem which would be solved through an optimization model. To add some context this is about a supply chain operational level planning decision(s) which almost always requires some kind of optimization.
Take the simplistic overview below (Figure 1). This diagram illustrates the virtual connection between the four "balloons" or aspects of a supply chain decision and very simply put it explains that to "deflate" one of the balloons, you need to "inflate" another. For example, if we need to minimize (deflate) the missed sales, we need to either increase our inventory, imporve our predicitability and forecasting, OR invest in our capacity for better responsiveness to any new demand requirement.

![Local Image](The4BalloonsofSupplyChain.png)
<div align="center">(Figure 1)</div>

Now let's review our physical flow (Figure 2) based on the problem scope. For the scope of this problem any order needs to be assigned to:
1. A plant (to make and ship the product in the order)
2. An origin sea port (as it seems the customers are all overseas customers
3. A specific sea freight carrier (contracted service)
4. A destination port (At which it seems like the ownership of the product is transitioned to the customer who assumes responsbility for it then)

Since the data format is available in .xlsx format. I have split it first into 7 csv files as required for this capstone project
But you can find find a brief overview of the data in (Figure 3) here.

![Local Image](ThePhysicalFlow.png)
<div align="center">(Figure 2)</div>

![Local Image](Data.png)
<div align="center">(Figure 3)</div>

## Objective function:
Since there is no specific objective function required for this problem but we are only provided with the above shared dataset, we need to make an assumption about what the business is requestimg based on the data provided. 
This could be one of the below likely objectives for such a problem:
1. Mazimize revenue
2. Minimize cost
3. Maxmize the number of fulfilled orders

The list can go on but given the data implying the importance of cost to be gathered and not providing any intel about the price (revenue), we will assume that the objective is minimizing cost

## Decision variables:
Again there is no one and only way to look at this problem or solve it, however from the data it appears that we need to decide for every order on a:
1. Plant/ WH
2. Origin Port
3. Sea freight carrier
4. Destination port

With the above, I decided to solve this problem as a Binary Integer Linear Problem where there are 9215x19x11x9 decision variables denoting a switch on or off for every order/plant/port/freight combination (respectively)


```python
! pip install openpyxl
! pip install plotly
! pip install pulp
import numpy as np
import pandas as pd
import networkx as nx
import matplotlib.pyplot as plt
import plotly.graph_objects as go
import seaborn as sns
import os
import plotly.graph_objects as go
from pulp import *
```

    Requirement already satisfied: openpyxl in c:\users\dell\anaconda3\lib\site-packages (3.0.5)
    Requirement already satisfied: jdcal in c:\users\dell\anaconda3\lib\site-packages (from openpyxl) (1.4.1)
    Requirement already satisfied: et-xmlfile in c:\users\dell\anaconda3\lib\site-packages (from openpyxl) (1.0.1)
    Requirement already satisfied: plotly in c:\users\dell\anaconda3\lib\site-packages (5.22.0)
    Requirement already satisfied: tenacity>=6.2.0 in c:\users\dell\anaconda3\lib\site-packages (from plotly) (8.3.0)
    Requirement already satisfied: packaging in c:\users\dell\appdata\roaming\python\python38\site-packages (from plotly) (24.0)
    Requirement already satisfied: pulp in c:\users\dell\anaconda3\lib\site-packages (2.8.0)
    

## Investigating the Optimisation Problem Constraints: 
### Constraint 1 (Plant-Port Links)

We will start with the feasible plant to port links and visualizing this data in a Sankey (Network) Diagram with nodes weighted by the number of links to emphasize the agility of each node supporting numerous source or destination nodes, for example if we have the below data:

| Plants | Ports  |
|--------|--------|
|   a    |     1  |
|   a    |     2  |
|   a    |     3  |
|   b    |     1  |
|   b    |     4  |

Plant a will have a score of 3 (connected to 3 ports) while Plant b has a score of 2. And similarly port 1 has a score of 2 while ports 2,3, and 4 have scores of 1.


```python
df = pd.read_csv('ImperialDACapstone/PlantPorts.csv')
df.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 22 entries, 0 to 21
    Data columns (total 2 columns):
     #   Column      Non-Null Count  Dtype 
    ---  ------      --------------  ----- 
     0   Plant Code  22 non-null     object
     1   Port        22 non-null     object
    dtypes: object(2)
    memory usage: 480.0+ bytes
    


```python
#Get the order number of supply chain nodes from their IDs to ease the creation of a list of same type facilities i.e. plant or port
def right2(s):
    return s[-2:]

plant_port = df.copy()
plant_port['Plant Code'] =  plant_port['Plant Code'].apply(right2).astype(int)
plant_port['Port'] =  plant_port['Port'].apply(right2).astype(int)
PlantsList = list(set(df['Plant Code']))
PlantsList.sort() #Get Sorted list of plants
PortsList = list(set(df['Port']))
PortsList.sort() #Get sorted list of ports
PlantsNPortsList = PlantsList+PortsList
#print(PlantsNPortsList)
plantPortsCount = df['Plant Code'].value_counts()
portPlantsCount = df['Port'].value_counts()
plant_port['PortIND'] = plant_port['Port']+len(PlantsList)

series_list = [plantPortsCount,portPlantsCount]
#print(series_list)
seriesConc = pd.concat(series_list)
#print(seriesConc)
plntNportLinksValues = []
for i in PlantsNPortsList:
    plntNportLinksValues.append(seriesConc[i])
    #print(i," ",seriesConc[i])
```


```python
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Plant Code</th>
      <th>Port</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>PLANT01</td>
      <td>PORT01</td>
    </tr>
    <tr>
      <th>1</th>
      <td>PLANT01</td>
      <td>PORT02</td>
    </tr>
    <tr>
      <th>2</th>
      <td>PLANT02</td>
      <td>PORT03</td>
    </tr>
    <tr>
      <th>3</th>
      <td>PLANT03</td>
      <td>PORT04</td>
    </tr>
    <tr>
      <th>4</th>
      <td>PLANT04</td>
      <td>PORT05</td>
    </tr>
  </tbody>
</table>
</div>




```python
all_nodes = plant_port['Plant Code'].tolist() + plant_port['PortIND'].tolist() ###
link_counts = {node: all_nodes.count(node) for node in set(all_nodes)} ###
max_links = max(link_counts.values()) ###
colors = plt.cm.viridis(np.linspace(0, 1, max_links)) ###
node_colors = [] ###
for label in PlantsNPortsList:###
    # Find the corresponding node index in all_nodes
    index = PlantsNPortsList.index(label) + 1  # +1 because node indices start from 1 ###
    node_colors.append(colors[link_counts.get(index, 0) - 1])  # -1 because colors are 0-indexed ###

# Convert colors to hex
node_colors = ['#%02x%02x%02x' % (int(c[0]*255), int(c[1]*255), int(c[2]*255)) for c in node_colors] ###

# Create legend annotations
legend_annotations = []
for i in range(max_links):
    color = colors[i]
    hex_color = '#%02x%02x%02x' % (int(color[0]*255), int(color[1]*255), int(color[2]*255))
    legend_annotations.append(
        dict(
            x=1.05, y=1 - i * 0.1,
            xref="paper", yref="paper",
            text=f"{i+1} link(s)",
            showarrow=False,
            font=dict(
                size=10,
                color=hex_color
            ),
            bgcolor="#FFFFFF",
            bordercolor=hex_color
        )
    )

fig = go.Figure(data=[go.Sankey(
    node = dict(
      pad = 15,
      thickness = 20,
      line = dict(color = "black", width = 0.5),
      label = PlantsNPortsList,
      color = node_colors
    ),
    link = dict(
      source = plant_port['Plant Code']-1, # indices correspond to labels, eg A1, A2, A1, B1, ...
      target = plant_port['PortIND']-1, value = plntNportLinksValues,
      label = PlantsList.append(PortsList)
  ))])

fig.update_layout(
    title_text="Basic Sankey Diagram of Plants to Ports Links",
    font_size=10,
    annotations=legend_annotations,
    margin=dict(l=50, r=200, t=50, b=50)
)

fig.show()# Heading 1
# Heading 2
## Heading 2.1
## Heading 2.2
print(list(set(plant_port['Plant Code'])))
print(list(set(plant_port['PortIND'])))
```


        <script type="text/javascript">
        window.PlotlyConfig = {MathJaxConfig: 'local'};
        if (window.MathJax && window.MathJax.Hub && window.MathJax.Hub.Config) {window.MathJax.Hub.Config({SVG: {font: "STIX-Web"}});}
        if (typeof require !== 'undefined') {
        require.undef("plotly");
        define('plotly', function(require, exports, module) {
            /**
* plotly.js v2.32.0
* Copyright 2012-2024, Plotly, Inc.
* All rights reserved.
* Licensed under the MIT license
*/
/*! For license information please see plotly.min.js.LICENSE.txt */
        });
        require(['plotly'], function(Plotly) {
            window._Plotly = Plotly;
        });
        }
        </script>




<div>                            <div id="e024326c-3b92-409a-8804-ed065b3da6ed" class="plotly-graph-div" style="height:525px; width:100%;"></div>            <script type="text/javascript">                require(["plotly"], function(Plotly) {                    window.PLOTLYENV=window.PLOTLYENV || {};                                    if (document.getElementById("e024326c-3b92-409a-8804-ed065b3da6ed")) {                    Plotly.newPlot(                        "e024326c-3b92-409a-8804-ed065b3da6ed",                        [{"link":{"source":[0,0,1,2,3,4,5,6,6,7,8,9,9,10,11,12,13,14,15,16,17,18],"target":[19,20,21,22,23,24,24,19,20,22,22,19,20,22,22,22,25,26,27,28,29,22],"value":[2,1,1,1,1,1,2,1,1,2,1,1,1,1,1,1,1,1,1,3,3,1,7,1,2,1,1,1,1,1]},"node":{"color":["#443982","#440154","#440154","#440154","#440154","#440154","#443982","#440154","#440154","#443982","#440154","#440154","#440154","#440154","#440154","#440154","#440154","#440154","#440154","#30678d","#30678d","#440154","#fde724","#440154","#443982","#440154","#440154","#440154","#440154","#440154"],"label":["PLANT01","PLANT02","PLANT03","PLANT04","PLANT05","PLANT06","PLANT07","PLANT08","PLANT09","PLANT10","PLANT11","PLANT12","PLANT13","PLANT14","PLANT15","PLANT16","PLANT17","PLANT18","PLANT19","PORT01","PORT02","PORT03","PORT04","PORT05","PORT06","PORT07","PORT08","PORT09","PORT10","PORT11"],"line":{"color":"black","width":0.5},"pad":15,"thickness":20},"type":"sankey"}],                        {"template":{"data":{"histogram2dcontour":[{"type":"histogram2dcontour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"choropleth":[{"type":"choropleth","colorbar":{"outlinewidth":0,"ticks":""}}],"histogram2d":[{"type":"histogram2d","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmap":[{"type":"heatmap","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmapgl":[{"type":"heatmapgl","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"contourcarpet":[{"type":"contourcarpet","colorbar":{"outlinewidth":0,"ticks":""}}],"contour":[{"type":"contour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"surface":[{"type":"surface","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"mesh3d":[{"type":"mesh3d","colorbar":{"outlinewidth":0,"ticks":""}}],"scatter":[{"fillpattern":{"fillmode":"overlay","size":10,"solidity":0.2},"type":"scatter"}],"parcoords":[{"type":"parcoords","line":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolargl":[{"type":"scatterpolargl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"bar":[{"error_x":{"color":"#2a3f5f"},"error_y":{"color":"#2a3f5f"},"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"bar"}],"scattergeo":[{"type":"scattergeo","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolar":[{"type":"scatterpolar","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"histogram":[{"marker":{"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"histogram"}],"scattergl":[{"type":"scattergl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatter3d":[{"type":"scatter3d","line":{"colorbar":{"outlinewidth":0,"ticks":""}},"marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattermapbox":[{"type":"scattermapbox","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterternary":[{"type":"scatterternary","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattercarpet":[{"type":"scattercarpet","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"carpet":[{"aaxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"baxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"type":"carpet"}],"table":[{"cells":{"fill":{"color":"#EBF0F8"},"line":{"color":"white"}},"header":{"fill":{"color":"#C8D4E3"},"line":{"color":"white"}},"type":"table"}],"barpolar":[{"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"barpolar"}],"pie":[{"automargin":true,"type":"pie"}]},"layout":{"autotypenumbers":"strict","colorway":["#636efa","#EF553B","#00cc96","#ab63fa","#FFA15A","#19d3f3","#FF6692","#B6E880","#FF97FF","#FECB52"],"font":{"color":"#2a3f5f"},"hovermode":"closest","hoverlabel":{"align":"left"},"paper_bgcolor":"white","plot_bgcolor":"#E5ECF6","polar":{"bgcolor":"#E5ECF6","angularaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"radialaxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"ternary":{"bgcolor":"#E5ECF6","aaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"baxis":{"gridcolor":"white","linecolor":"white","ticks":""},"caxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"coloraxis":{"colorbar":{"outlinewidth":0,"ticks":""}},"colorscale":{"sequential":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"sequentialminus":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"diverging":[[0,"#8e0152"],[0.1,"#c51b7d"],[0.2,"#de77ae"],[0.3,"#f1b6da"],[0.4,"#fde0ef"],[0.5,"#f7f7f7"],[0.6,"#e6f5d0"],[0.7,"#b8e186"],[0.8,"#7fbc41"],[0.9,"#4d9221"],[1,"#276419"]]},"xaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"yaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"scene":{"xaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"yaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"zaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2}},"shapedefaults":{"line":{"color":"#2a3f5f"}},"annotationdefaults":{"arrowcolor":"#2a3f5f","arrowhead":0,"arrowwidth":1},"geo":{"bgcolor":"white","landcolor":"#E5ECF6","subunitcolor":"white","showland":true,"showlakes":true,"lakecolor":"white"},"title":{"x":0.05},"mapbox":{"style":"light"}}},"title":{"text":"Basic Sankey Diagram of Plants to Ports Links"},"font":{"size":10},"margin":{"l":50,"r":200,"t":50,"b":50},"annotations":[{"bgcolor":"#FFFFFF","bordercolor":"#440154","font":{"color":"#440154","size":10},"showarrow":false,"text":"1 link(s)","x":1.05,"xref":"paper","y":1.0,"yref":"paper"},{"bgcolor":"#FFFFFF","bordercolor":"#443982","font":{"color":"#443982","size":10},"showarrow":false,"text":"2 link(s)","x":1.05,"xref":"paper","y":0.9,"yref":"paper"},{"bgcolor":"#FFFFFF","bordercolor":"#30678d","font":{"color":"#30678d","size":10},"showarrow":false,"text":"3 link(s)","x":1.05,"xref":"paper","y":0.8,"yref":"paper"},{"bgcolor":"#FFFFFF","bordercolor":"#20908c","font":{"color":"#20908c","size":10},"showarrow":false,"text":"4 link(s)","x":1.05,"xref":"paper","y":0.7,"yref":"paper"},{"bgcolor":"#FFFFFF","bordercolor":"#35b778","font":{"color":"#35b778","size":10},"showarrow":false,"text":"5 link(s)","x":1.05,"xref":"paper","y":0.6,"yref":"paper"},{"bgcolor":"#FFFFFF","bordercolor":"#90d643","font":{"color":"#90d643","size":10},"showarrow":false,"text":"6 link(s)","x":1.05,"xref":"paper","y":0.5,"yref":"paper"},{"bgcolor":"#FFFFFF","bordercolor":"#fde724","font":{"color":"#fde724","size":10},"showarrow":false,"text":"7 link(s)","x":1.05,"xref":"paper","y":0.3999999999999999,"yref":"paper"}]},                        {"responsive": true}                    ).then(function(){

var gd = document.getElementById('e024326c-3b92-409a-8804-ed065b3da6ed');
var x = new MutationObserver(function (mutations, observer) {{
        var display = window.getComputedStyle(gd).display;
        if (!display || display === 'none') {{
            console.log([gd, 'removed!']);
            Plotly.purge(gd);
            observer.disconnect();
        }}
}});

// Listen for the removal of the full notebook cells
var notebookContainer = gd.closest('#notebook-container');
if (notebookContainer) {{
    x.observe(notebookContainer, {childList: true});
}}

// Listen for the clearing of the current output cell
var outputEl = gd.closest('.output');
if (outputEl) {{
    x.observe(outputEl, {childList: true});
}}

                        })                };                });            </script>        </div>


    [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19]
    [20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30]
    

### Constraint 2 (Port-Freight Supported Links)

Similarly, We will start with the feasible port-freight combinations and visualizing this data in a Sankey (Network) Diagram with nodes weighted by the number of links to emphasize the agility of each node.


```python
df = pd.read_csv('ImperialDACapstone/FreightRates.csv')
df.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 9215 entries, 0 to 9214
    Data columns (total 14 columns):
     #   Column        Non-Null Count  Dtype  
    ---  ------        --------------  -----  
     0   Carrier       1540 non-null   object 
     1   orig_port_cd  1540 non-null   object 
     2   dest_port_cd  1540 non-null   object 
     3   minm_wgh_qty  1540 non-null   float64
     4   max_wgh_qty   1540 non-null   object 
     5   svc_cd        1540 non-null   object 
     6   minimum cost  1540 non-null   object 
     7   rate          1540 non-null   object 
     8   mode_dsc      1540 non-null   object 
     9   tpt_day_cnt   1540 non-null   float64
     10  Carrier type  1540 non-null   object 
     11  Unnamed: 11   0 non-null      float64
     12  Unnamed: 12   0 non-null      float64
     13  Unnamed: 13   0 non-null      float64
    dtypes: float64(5), object(9)
    memory usage: 1008.0+ KB
    


```python
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Carrier</th>
      <th>orig_port_cd</th>
      <th>dest_port_cd</th>
      <th>minm_wgh_qty</th>
      <th>max_wgh_qty</th>
      <th>svc_cd</th>
      <th>minimum cost</th>
      <th>rate</th>
      <th>mode_dsc</th>
      <th>tpt_day_cnt</th>
      <th>Carrier type</th>
      <th>Unnamed: 11</th>
      <th>Unnamed: 12</th>
      <th>Unnamed: 13</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>V444_6</td>
      <td>PORT08</td>
      <td>PORT09</td>
      <td>250.0</td>
      <td>499.99</td>
      <td>DTD</td>
      <td>$43.23</td>
      <td>$0.71</td>
      <td>AIR</td>
      <td>2.0</td>
      <td>V88888888_0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>1</th>
      <td>V444_6</td>
      <td>PORT08</td>
      <td>PORT09</td>
      <td>65.0</td>
      <td>69.99</td>
      <td>DTD</td>
      <td>$43.23</td>
      <td>$0.75</td>
      <td>AIR</td>
      <td>2.0</td>
      <td>V88888888_0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2</th>
      <td>V444_6</td>
      <td>PORT08</td>
      <td>PORT09</td>
      <td>60.0</td>
      <td>64.99</td>
      <td>DTD</td>
      <td>$43.23</td>
      <td>$0.79</td>
      <td>AIR</td>
      <td>2.0</td>
      <td>V88888888_0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>3</th>
      <td>V444_6</td>
      <td>PORT08</td>
      <td>PORT09</td>
      <td>50.0</td>
      <td>54.99</td>
      <td>DTD</td>
      <td>$43.23</td>
      <td>$0.83</td>
      <td>AIR</td>
      <td>2.0</td>
      <td>V88888888_0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>4</th>
      <td>V444_6</td>
      <td>PORT08</td>
      <td>PORT09</td>
      <td>35.0</td>
      <td>39.99</td>
      <td>DTD</td>
      <td>$43.23</td>
      <td>$1.06</td>
      <td>AIR</td>
      <td>2.0</td>
      <td>V88888888_0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Count values in column 'dest_port_cd'
value_counts_Dest = df['dest_port_cd'].value_counts()
# Count unique freight values in column 'dest_port_cd'
value_counts_Dest_U = df.groupby('dest_port_cd')['Carrier'].nunique()
# Count values in column 'orig_port_cd'
value_counts_Origin = df['orig_port_cd'].value_counts()
# Count unique freight values in column 'orig_port_cd'
value_counts_Origin_U = df.groupby('orig_port_cd')['Carrier'].nunique()

print("Value counts for Destination Ports:")
print(value_counts_Dest_U)

print("\nValue counts for Origin Ports:")
print(value_counts_Origin_U)
```

    Value counts for Destination Ports:
    dest_port_cd
    PORT09    9
    Name: Carrier, dtype: int64
    
    Value counts for Origin Ports:
    orig_port_cd
    PORT02    6
    PORT03    4
    PORT04    4
    PORT05    3
    PORT06    3
    PORT07    1
    PORT08    3
    PORT09    3
    PORT10    4
    PORT11    2
    Name: Carrier, dtype: int64
    


```python
flow_counts = df.groupby(['orig_port_cd', 'Carrier']).size().reset_index(name='Count')
```


```python
# Get unique labels
all_labels = list(pd.concat([df['orig_port_cd'], df['Carrier']]).unique())

# Create a dictionary to map category names to indices
label_indices = {label: idx for idx, label in enumerate(all_labels)}

# Map the source and target categories to indices
flow_counts['SourceIndex'] = flow_counts['orig_port_cd'].map(label_indices)
flow_counts['TargetIndex'] = flow_counts['Carrier'].map(label_indices)

# Create the source, target, and value lists
source_indices = flow_counts['SourceIndex'].tolist()
target_indices = flow_counts['TargetIndex'].tolist()
values = flow_counts['Count'].tolist()
```


```python
fig = go.Figure(data=[go.Sankey(
    node=dict(
        pad=15,
        thickness=20,
        line=dict(color="black", width=0.5),
        label=all_labels
    ),
    link=dict(
        source=source_indices,
        target=target_indices,
        value=values
    )
)])

fig.update_layout(title_text="Sankey Diagram", font_size=10)
fig.show()
```


<div>                            <div id="d1778fca-c3ef-42c5-963e-23e2a5ccf69e" class="plotly-graph-div" style="height:525px; width:100%;"></div>            <script type="text/javascript">                require(["plotly"], function(Plotly) {                    window.PLOTLYENV=window.PLOTLYENV || {};                                    if (document.getElementById("d1778fca-c3ef-42c5-963e-23e2a5ccf69e")) {                    Plotly.newPlot(                        "d1778fca-c3ef-42c5-963e-23e2a5ccf69e",                        [{"link":{"source":[5,5,5,5,5,5,6,6,6,6,4,4,4,4,8,8,8,9,9,9,7,0,0,0,2,2,2,1,1,1,1,3,3],"target":[16,15,14,18,17,12,16,17,19,12,16,15,18,12,15,18,17,15,18,17,14,14,11,13,16,17,12,15,14,18,11,14,12],"value":[5,20,5,151,25,20,12,12,1,20,20,40,151,20,48,151,20,152,302,25,20,15,19,5,2,2,20,32,20,151,19,15,20]},"node":{"label":["PORT08","PORT10","PORT09","PORT11","PORT04","PORT02","PORT03","PORT07","PORT05","PORT06",null,"V444_6","V444_8","V444_9","V444_2","V444_1","V444_0","V444_5","V444_4","V444_7"],"line":{"color":"black","width":0.5},"pad":15,"thickness":20},"type":"sankey"}],                        {"template":{"data":{"histogram2dcontour":[{"type":"histogram2dcontour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"choropleth":[{"type":"choropleth","colorbar":{"outlinewidth":0,"ticks":""}}],"histogram2d":[{"type":"histogram2d","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmap":[{"type":"heatmap","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmapgl":[{"type":"heatmapgl","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"contourcarpet":[{"type":"contourcarpet","colorbar":{"outlinewidth":0,"ticks":""}}],"contour":[{"type":"contour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"surface":[{"type":"surface","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"mesh3d":[{"type":"mesh3d","colorbar":{"outlinewidth":0,"ticks":""}}],"scatter":[{"fillpattern":{"fillmode":"overlay","size":10,"solidity":0.2},"type":"scatter"}],"parcoords":[{"type":"parcoords","line":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolargl":[{"type":"scatterpolargl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"bar":[{"error_x":{"color":"#2a3f5f"},"error_y":{"color":"#2a3f5f"},"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"bar"}],"scattergeo":[{"type":"scattergeo","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolar":[{"type":"scatterpolar","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"histogram":[{"marker":{"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"histogram"}],"scattergl":[{"type":"scattergl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatter3d":[{"type":"scatter3d","line":{"colorbar":{"outlinewidth":0,"ticks":""}},"marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattermapbox":[{"type":"scattermapbox","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterternary":[{"type":"scatterternary","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattercarpet":[{"type":"scattercarpet","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"carpet":[{"aaxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"baxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"type":"carpet"}],"table":[{"cells":{"fill":{"color":"#EBF0F8"},"line":{"color":"white"}},"header":{"fill":{"color":"#C8D4E3"},"line":{"color":"white"}},"type":"table"}],"barpolar":[{"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"barpolar"}],"pie":[{"automargin":true,"type":"pie"}]},"layout":{"autotypenumbers":"strict","colorway":["#636efa","#EF553B","#00cc96","#ab63fa","#FFA15A","#19d3f3","#FF6692","#B6E880","#FF97FF","#FECB52"],"font":{"color":"#2a3f5f"},"hovermode":"closest","hoverlabel":{"align":"left"},"paper_bgcolor":"white","plot_bgcolor":"#E5ECF6","polar":{"bgcolor":"#E5ECF6","angularaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"radialaxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"ternary":{"bgcolor":"#E5ECF6","aaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"baxis":{"gridcolor":"white","linecolor":"white","ticks":""},"caxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"coloraxis":{"colorbar":{"outlinewidth":0,"ticks":""}},"colorscale":{"sequential":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"sequentialminus":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"diverging":[[0,"#8e0152"],[0.1,"#c51b7d"],[0.2,"#de77ae"],[0.3,"#f1b6da"],[0.4,"#fde0ef"],[0.5,"#f7f7f7"],[0.6,"#e6f5d0"],[0.7,"#b8e186"],[0.8,"#7fbc41"],[0.9,"#4d9221"],[1,"#276419"]]},"xaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"yaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"scene":{"xaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"yaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"zaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2}},"shapedefaults":{"line":{"color":"#2a3f5f"}},"annotationdefaults":{"arrowcolor":"#2a3f5f","arrowhead":0,"arrowwidth":1},"geo":{"bgcolor":"white","landcolor":"#E5ECF6","subunitcolor":"white","showland":true,"showlakes":true,"lakecolor":"white"},"title":{"x":0.05},"mapbox":{"style":"light"}}},"title":{"text":"Sankey Diagram"},"font":{"size":10}},                        {"responsive": true}                    ).then(function(){

var gd = document.getElementById('d1778fca-c3ef-42c5-963e-23e2a5ccf69e');
var x = new MutationObserver(function (mutations, observer) {{
        var display = window.getComputedStyle(gd).display;
        if (!display || display === 'none') {{
            console.log([gd, 'removed!']);
            Plotly.purge(gd);
            observer.disconnect();
        }}
}});

// Listen for the removal of the full notebook cells
var notebookContainer = gd.closest('#notebook-container');
if (notebookContainer) {{
    x.observe(notebookContainer, {childList: true});
}}

// Listen for the clearing of the current output cell
var outputEl = gd.closest('.output');
if (outputEl) {{
    x.observe(outputEl, {childList: true});
}}

                        })                };                });            </script>        </div>


From the above, we can observe that:
1. There is only one destination port (PORT09) so we don't need to "decide" on the destination port 
2. Ports 6,4,2, 10 and 5 seem to have the most links with different freights
3. Although Port 1 seemed to have the highest links to plants from studying the previous constraint, it seems to not be supported by any of our contracted freights *or the data is missing!*

### Constraint 3 (Products-Plant)

This is to investigate all the supported products/ plants combinations and add a comparative analysis with the products list in the orders we need to fulfill


```python
dfProdPlant = pd.read_csv('ImperialDACapstone/ProductsPerPlant.csv')
dfProdPlant.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 9215 entries, 0 to 9214
    Data columns (total 14 columns):
     #   Column       Non-Null Count  Dtype  
    ---  ------       --------------  -----  
     0   Plant Code   2036 non-null   object 
     1   Product ID   2036 non-null   float64
     2   Unnamed: 2   0 non-null      float64
     3   Unnamed: 3   0 non-null      float64
     4   Unnamed: 4   0 non-null      float64
     5   Unnamed: 5   0 non-null      float64
     6   Unnamed: 6   0 non-null      float64
     7   Unnamed: 7   0 non-null      float64
     8   Unnamed: 8   0 non-null      float64
     9   Unnamed: 9   0 non-null      float64
     10  Unnamed: 10  0 non-null      float64
     11  Unnamed: 11  0 non-null      float64
     12  Unnamed: 12  0 non-null      float64
     13  Unnamed: 13  0 non-null      float64
    dtypes: float64(13), object(1)
    memory usage: 1008.0+ KB
    


```python
dfProdPlant.columns = [col.strip().replace(' ', '_').replace('/', '_') for col in dfProdPlant.columns]

dfProdPlant = dfProdPlant[['Plant_Code','Product_ID']]
dfProdPlant.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Plant_Code</th>
      <th>Product_ID</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>PLANT15</td>
      <td>1698815.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>PLANT17</td>
      <td>1664419.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>PLANT17</td>
      <td>1664426.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>PLANT17</td>
      <td>1672826.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>PLANT17</td>
      <td>1674916.0</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Get the products list in the orders we need to plan fulfilment for
df_orders = pd.read_csv('ImperialDACapstone/OrderList.csv')
df_orders.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 9215 entries, 0 to 9214
    Data columns (total 14 columns):
     #   Column                Non-Null Count  Dtype  
    ---  ------                --------------  -----  
     0   Order ID              9215 non-null   int64  
     1   Order Date            9215 non-null   object 
     2   Origin Port           9215 non-null   object 
     3   Carrier               9215 non-null   object 
     4   TPT                   9215 non-null   int64  
     5   Service Level         9215 non-null   object 
     6   Ship ahead day count  9215 non-null   int64  
     7   Ship Late Day count   9215 non-null   int64  
     8   Customer              9215 non-null   object 
     9   Product ID            9215 non-null   int64  
     10  Plant Code            9215 non-null   object 
     11  Destination Port      9215 non-null   object 
     12  Unit quantity         9215 non-null   int64  
     13  Weight                9215 non-null   float64
    dtypes: float64(1), int64(6), object(7)
    memory usage: 1008.0+ KB
    


```python
df_orders.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Order ID</th>
      <th>Order Date</th>
      <th>Origin Port</th>
      <th>Carrier</th>
      <th>TPT</th>
      <th>Service Level</th>
      <th>Ship ahead day count</th>
      <th>Ship Late Day count</th>
      <th>Customer</th>
      <th>Product ID</th>
      <th>Plant Code</th>
      <th>Destination Port</th>
      <th>Unit quantity</th>
      <th>Weight</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1447296447</td>
      <td>26-May</td>
      <td>PORT09</td>
      <td>V44_3</td>
      <td>1</td>
      <td>CRF</td>
      <td>3</td>
      <td>0</td>
      <td>V55555_53</td>
      <td>1700106</td>
      <td>PLANT16</td>
      <td>PORT09</td>
      <td>808</td>
      <td>14.30</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1447158015</td>
      <td>26-May</td>
      <td>PORT09</td>
      <td>V44_3</td>
      <td>1</td>
      <td>CRF</td>
      <td>3</td>
      <td>0</td>
      <td>V55555_53</td>
      <td>1700106</td>
      <td>PLANT16</td>
      <td>PORT09</td>
      <td>3188</td>
      <td>87.94</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1447138899</td>
      <td>26-May</td>
      <td>PORT09</td>
      <td>V44_3</td>
      <td>1</td>
      <td>CRF</td>
      <td>3</td>
      <td>0</td>
      <td>V55555_53</td>
      <td>1700106</td>
      <td>PLANT16</td>
      <td>PORT09</td>
      <td>2331</td>
      <td>61.20</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1447363528</td>
      <td>26-May</td>
      <td>PORT09</td>
      <td>V44_3</td>
      <td>1</td>
      <td>CRF</td>
      <td>3</td>
      <td>0</td>
      <td>V55555_53</td>
      <td>1700106</td>
      <td>PLANT16</td>
      <td>PORT09</td>
      <td>847</td>
      <td>16.16</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1447363981</td>
      <td>26-May</td>
      <td>PORT09</td>
      <td>V44_3</td>
      <td>1</td>
      <td>CRF</td>
      <td>3</td>
      <td>0</td>
      <td>V55555_53</td>
      <td>1700106</td>
      <td>PLANT16</td>
      <td>PORT09</td>
      <td>2163</td>
      <td>52.34</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Get the # of orders per product
df_orders.columns = [col.strip().replace(' ', '_').replace('/', '_') for col in df_orders.columns]
df_orders.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Order_ID</th>
      <th>Order_Date</th>
      <th>Origin_Port</th>
      <th>Carrier</th>
      <th>TPT</th>
      <th>Service_Level</th>
      <th>Ship_ahead_day_count</th>
      <th>Ship_Late_Day_count</th>
      <th>Customer</th>
      <th>Product_ID</th>
      <th>Plant_Code</th>
      <th>Destination_Port</th>
      <th>Unit_quantity</th>
      <th>Weight</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1447296447</td>
      <td>26-May</td>
      <td>PORT09</td>
      <td>V44_3</td>
      <td>1</td>
      <td>CRF</td>
      <td>3</td>
      <td>0</td>
      <td>V55555_53</td>
      <td>1700106</td>
      <td>PLANT16</td>
      <td>PORT09</td>
      <td>808</td>
      <td>14.30</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1447158015</td>
      <td>26-May</td>
      <td>PORT09</td>
      <td>V44_3</td>
      <td>1</td>
      <td>CRF</td>
      <td>3</td>
      <td>0</td>
      <td>V55555_53</td>
      <td>1700106</td>
      <td>PLANT16</td>
      <td>PORT09</td>
      <td>3188</td>
      <td>87.94</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1447138899</td>
      <td>26-May</td>
      <td>PORT09</td>
      <td>V44_3</td>
      <td>1</td>
      <td>CRF</td>
      <td>3</td>
      <td>0</td>
      <td>V55555_53</td>
      <td>1700106</td>
      <td>PLANT16</td>
      <td>PORT09</td>
      <td>2331</td>
      <td>61.20</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1447363528</td>
      <td>26-May</td>
      <td>PORT09</td>
      <td>V44_3</td>
      <td>1</td>
      <td>CRF</td>
      <td>3</td>
      <td>0</td>
      <td>V55555_53</td>
      <td>1700106</td>
      <td>PLANT16</td>
      <td>PORT09</td>
      <td>847</td>
      <td>16.16</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1447363981</td>
      <td>26-May</td>
      <td>PORT09</td>
      <td>V44_3</td>
      <td>1</td>
      <td>CRF</td>
      <td>3</td>
      <td>0</td>
      <td>V55555_53</td>
      <td>1700106</td>
      <td>PLANT16</td>
      <td>PORT09</td>
      <td>2163</td>
      <td>52.34</td>
    </tr>
  </tbody>
</table>
</div>




```python
value_counts_OrdersPerProduct = df_orders.groupby('Product_ID')['Order_ID'].nunique()
# Converting to a pandas dataframe
value_counts_OrdersPerProduct_df = value_counts_OrdersPerProduct.reset_index(name='unique_count')
# Sorting
sorted_df = value_counts_OrdersPerProduct_df.sort_values(by='unique_count', ascending=False)

print("\nValue counts for Orders per Product:")
print(sorted_df)
```

    
    Value counts for Orders per Product:
         Product_ID  unique_count
    449     1689547           192
    250     1677878           140
    450     1689548           133
    448     1689546           129
    419     1688571           120
    ..          ...           ...
    270     1680246             1
    267     1679995             1
    266     1679991             1
    624     1699290             1
    262     1678759             1
    
    [772 rows x 2 columns]
    


```python
sorted_df['unique_count'].sum()
```




    9215




```python
# Allow for an interactive histogram with changing bins
!pip install ipywidgets matplotlib ipython
```

    Requirement already satisfied: ipywidgets in c:\users\dell\anaconda3\lib\site-packages (7.5.1)
    Requirement already satisfied: matplotlib in c:\users\dell\anaconda3\lib\site-packages (3.3.2)
    Requirement already satisfied: ipython in c:\users\dell\anaconda3\lib\site-packages (7.19.0)
    Requirement already satisfied: widgetsnbextension~=3.5.0 in c:\users\dell\anaconda3\lib\site-packages (from ipywidgets) (3.5.1)
    Requirement already satisfied: ipykernel>=4.5.1 in c:\users\dell\anaconda3\lib\site-packages (from ipywidgets) (5.3.4)
    Requirement already satisfied: traitlets>=4.3.1 in c:\users\dell\anaconda3\lib\site-packages (from ipywidgets) (5.0.5)
    Requirement already satisfied: nbformat>=4.2.0 in c:\users\dell\anaconda3\lib\site-packages (from ipywidgets) (5.0.8)
    Requirement already satisfied: cycler>=0.10 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib) (0.10.0)
    Requirement already satisfied: python-dateutil>=2.1 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib) (2.8.1)
    Requirement already satisfied: kiwisolver>=1.0.1 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib) (1.3.0)
    Requirement already satisfied: numpy>=1.15 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib) (1.24.4)
    Requirement already satisfied: certifi>=2020.06.20 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib) (2020.6.20)
    Requirement already satisfied: pyparsing!=2.0.4,!=2.1.2,!=2.1.6,>=2.0.3 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib) (2.4.7)
    Requirement already satisfied: pillow>=6.2.0 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib) (8.0.1)
    Requirement already satisfied: backcall in c:\users\dell\anaconda3\lib\site-packages (from ipython) (0.2.0)
    Requirement already satisfied: jedi>=0.10 in c:\users\dell\anaconda3\lib\site-packages (from ipython) (0.17.1)
    Requirement already satisfied: decorator in c:\users\dell\anaconda3\lib\site-packages (from ipython) (4.4.2)
    Requirement already satisfied: pygments in c:\users\dell\anaconda3\lib\site-packages (from ipython) (2.7.2)
    Requirement already satisfied: prompt-toolkit!=3.0.0,!=3.0.1,<3.1.0,>=2.0.0 in c:\users\dell\anaconda3\lib\site-packages (from ipython) (3.0.8)
    Requirement already satisfied: pickleshare in c:\users\dell\anaconda3\lib\site-packages (from ipython) (0.7.5)
    Requirement already satisfied: colorama; sys_platform == "win32" in c:\users\dell\anaconda3\lib\site-packages (from ipython) (0.4.4)
    Requirement already satisfied: setuptools>=18.5 in c:\users\dell\anaconda3\lib\site-packages (from ipython) (50.3.1.post20201107)
    Requirement already satisfied: notebook>=4.4.1 in c:\users\dell\anaconda3\lib\site-packages (from widgetsnbextension~=3.5.0->ipywidgets) (6.1.4)
    Requirement already satisfied: tornado>=4.2 in c:\users\dell\anaconda3\lib\site-packages (from ipykernel>=4.5.1->ipywidgets) (6.0.4)
    Requirement already satisfied: jupyter-client in c:\users\dell\anaconda3\lib\site-packages (from ipykernel>=4.5.1->ipywidgets) (6.1.7)
    Requirement already satisfied: ipython-genutils in c:\users\dell\anaconda3\lib\site-packages (from traitlets>=4.3.1->ipywidgets) (0.2.0)
    Requirement already satisfied: jupyter-core in c:\users\dell\anaconda3\lib\site-packages (from nbformat>=4.2.0->ipywidgets) (4.6.3)
    Requirement already satisfied: jsonschema!=2.5.0,>=2.4 in c:\users\dell\anaconda3\lib\site-packages (from nbformat>=4.2.0->ipywidgets) (3.2.0)
    Requirement already satisfied: six in c:\users\dell\anaconda3\lib\site-packages (from cycler>=0.10->matplotlib) (1.15.0)
    Requirement already satisfied: parso<0.8.0,>=0.7.0 in c:\users\dell\anaconda3\lib\site-packages (from jedi>=0.10->ipython) (0.7.0)
    Requirement already satisfied: wcwidth in c:\users\dell\anaconda3\lib\site-packages (from prompt-toolkit!=3.0.0,!=3.0.1,<3.1.0,>=2.0.0->ipython) (0.2.5)
    Requirement already satisfied: terminado>=0.8.3 in c:\users\dell\anaconda3\lib\site-packages (from notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (0.9.1)
    Requirement already satisfied: nbconvert in c:\users\dell\anaconda3\lib\site-packages (from notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (6.0.7)
    Requirement already satisfied: prometheus-client in c:\users\dell\anaconda3\lib\site-packages (from notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (0.8.0)
    Requirement already satisfied: jinja2 in c:\users\dell\anaconda3\lib\site-packages (from notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (2.11.2)
    Requirement already satisfied: pyzmq>=17 in c:\users\dell\anaconda3\lib\site-packages (from notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (19.0.2)
    Requirement already satisfied: Send2Trash in c:\users\dell\anaconda3\lib\site-packages (from notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (1.5.0)
    Requirement already satisfied: argon2-cffi in c:\users\dell\anaconda3\lib\site-packages (from notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (20.1.0)
    Requirement already satisfied: pywin32>=1.0; sys_platform == "win32" in c:\users\dell\anaconda3\lib\site-packages (from jupyter-core->nbformat>=4.2.0->ipywidgets) (227)
    Requirement already satisfied: attrs>=17.4.0 in c:\users\dell\anaconda3\lib\site-packages (from jsonschema!=2.5.0,>=2.4->nbformat>=4.2.0->ipywidgets) (20.3.0)
    Requirement already satisfied: pyrsistent>=0.14.0 in c:\users\dell\anaconda3\lib\site-packages (from jsonschema!=2.5.0,>=2.4->nbformat>=4.2.0->ipywidgets) (0.17.3)
    Requirement already satisfied: pywinpty>=0.5 in c:\users\dell\anaconda3\lib\site-packages (from terminado>=0.8.3->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (0.5.7)
    Requirement already satisfied: jupyterlab-pygments in c:\users\dell\anaconda3\lib\site-packages (from nbconvert->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (0.1.2)
    Requirement already satisfied: bleach in c:\users\dell\anaconda3\lib\site-packages (from nbconvert->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (3.2.1)
    Requirement already satisfied: pandocfilters>=1.4.1 in c:\users\dell\anaconda3\lib\site-packages (from nbconvert->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (1.4.3)
    Requirement already satisfied: testpath in c:\users\dell\anaconda3\lib\site-packages (from nbconvert->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (0.4.4)
    Requirement already satisfied: nbclient<0.6.0,>=0.5.0 in c:\users\dell\anaconda3\lib\site-packages (from nbconvert->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (0.5.1)
    Requirement already satisfied: defusedxml in c:\users\dell\anaconda3\lib\site-packages (from nbconvert->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (0.6.0)
    Requirement already satisfied: entrypoints>=0.2.2 in c:\users\dell\anaconda3\lib\site-packages (from nbconvert->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (0.3)
    Requirement already satisfied: mistune<2,>=0.8.1 in c:\users\dell\anaconda3\lib\site-packages (from nbconvert->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (0.8.4)
    Requirement already satisfied: MarkupSafe>=0.23 in c:\users\dell\anaconda3\lib\site-packages (from jinja2->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (1.1.1)
    Requirement already satisfied: cffi>=1.0.0 in c:\users\dell\anaconda3\lib\site-packages (from argon2-cffi->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (1.14.3)
    Requirement already satisfied: webencodings in c:\users\dell\anaconda3\lib\site-packages (from bleach->nbconvert->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (0.5.1)
    Requirement already satisfied: packaging in c:\users\dell\appdata\roaming\python\python38\site-packages (from bleach->nbconvert->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (24.0)
    Requirement already satisfied: async-generator in c:\users\dell\anaconda3\lib\site-packages (from nbclient<0.6.0,>=0.5.0->nbconvert->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (1.10)
    Requirement already satisfied: nest-asyncio in c:\users\dell\anaconda3\lib\site-packages (from nbclient<0.6.0,>=0.5.0->nbconvert->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (1.4.2)
    Requirement already satisfied: pycparser in c:\users\dell\anaconda3\lib\site-packages (from cffi>=1.0.0->argon2-cffi->notebook>=4.4.1->widgetsnbextension~=3.5.0->ipywidgets) (2.20)
    


```python
from ipywidgets import interact, IntSlider
from IPython.display import display
```


```python
!pip install mplcursors
```

    Requirement already satisfied: mplcursors in c:\users\dell\anaconda3\lib\site-packages (0.5.3)
    Requirement already satisfied: matplotlib!=3.7.1,>=3.1 in c:\users\dell\anaconda3\lib\site-packages (from mplcursors) (3.3.2)
    Requirement already satisfied: python-dateutil>=2.1 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib!=3.7.1,>=3.1->mplcursors) (2.8.1)
    Requirement already satisfied: certifi>=2020.06.20 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib!=3.7.1,>=3.1->mplcursors) (2020.6.20)
    Requirement already satisfied: pillow>=6.2.0 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib!=3.7.1,>=3.1->mplcursors) (8.0.1)
    Requirement already satisfied: cycler>=0.10 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib!=3.7.1,>=3.1->mplcursors) (0.10.0)
    Requirement already satisfied: pyparsing!=2.0.4,!=2.1.2,!=2.1.6,>=2.0.3 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib!=3.7.1,>=3.1->mplcursors) (2.4.7)
    Requirement already satisfied: kiwisolver>=1.0.1 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib!=3.7.1,>=3.1->mplcursors) (1.3.0)
    Requirement already satisfied: numpy>=1.15 in c:\users\dell\anaconda3\lib\site-packages (from matplotlib!=3.7.1,>=3.1->mplcursors) (1.24.4)
    Requirement already satisfied: six>=1.5 in c:\users\dell\anaconda3\lib\site-packages (from python-dateutil>=2.1->matplotlib!=3.7.1,>=3.1->mplcursors) (1.15.0)
    


```python
import mplcursors
```


```python
# Create a histogram of # of orders/ product distribution
# Plot histogram of column 'unique_column'
# Function to plot histogram with a given number of bins
def plot_histogram(bins):
    plt.figure(figsize=(15, 10))
    # n, bins, patches = ax.hist(sorted_df['unique_count'], bins=bins, edgecolor='black', color='skyblue', alpha=0.7)
    plt.hist(sorted_df['unique_count'], bins=bins, edgecolor='black', color='skyblue', alpha=0.7)
    plt.title('Histogram of # of orders per product')
    plt.xlabel('Value')
    plt.ylabel('Frequency')
    # Add cursor for interactive data points
    cursor = mplcursors.cursor(hover=False)
    cursor.connect("add", lambda sel: sel.annotation.set_text(f'Value: {sel.target[0]}\nFrequency: {sel.target[1]}'))
    plt.show()


# Interactive slider for bin size
bins_slider = IntSlider(value=5, min=1, max=20, step=1, description='Bins:')


# Create and show the interactive plot
interact(plot_histogram, bins=bins_slider)
```


    interactive(children=(IntSlider(value=5, description='Bins:', max=20, min=1), Output()), _dom_classes=('widget…





    <function __main__.plot_histogram(bins)>



The above investigates the number of orders per product ID, however, we still need to assess this versus supported plant/ product combinations i.e. plants feasibility to cover orders


```python
# We are choosing right join as we are only interested in products within the order list
mergedProdPlantOrd = dfProdPlant.merge(sorted_df, on='Product_ID',how='right')
mergedProdPlantOrd.head()
```

    C:\Users\dell\anaconda3\lib\site-packages\pandas\core\reshape\merge.py:1112: RuntimeWarning:
    
    invalid value encountered in cast
    
    




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Plant_Code</th>
      <th>Product_ID</th>
      <th>unique_count</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>PLANT02</td>
      <td>1689547.0</td>
      <td>192</td>
    </tr>
    <tr>
      <th>1</th>
      <td>PLANT10</td>
      <td>1689547.0</td>
      <td>192</td>
    </tr>
    <tr>
      <th>2</th>
      <td>PLANT11</td>
      <td>1689547.0</td>
      <td>192</td>
    </tr>
    <tr>
      <th>3</th>
      <td>PLANT03</td>
      <td>1689547.0</td>
      <td>192</td>
    </tr>
    <tr>
      <th>4</th>
      <td>PLANT03</td>
      <td>1677878.0</td>
      <td>140</td>
    </tr>
  </tbody>
</table>
</div>




```python
df_feasible_OrdPerPlant = mergedProdPlantOrd.groupby('Plant_Code')['unique_count'].sum().reset_index()
sorted_feasible_OrdPerPlant_df = df_feasible_OrdPerPlant.sort_values(by='unique_count', ascending=False)
print("\nTotal Feasible Orders Per Plant:")
print(sorted_feasible_OrdPerPlant_df)
```

    
    Total Feasible Orders Per Plant:
       Plant_Code  unique_count
    2     PLANT03          8600
    1     PLANT02          2104
    9     PLANT10          2060
    10    PLANT11          1995
    3     PLANT04           387
    11    PLANT12           302
    4     PLANT05           199
    14    PLANT16           174
    7     PLANT08           103
    6     PLANT07           102
    12    PLANT13            98
    8     PLANT09            74
    5     PLANT06            21
    15    PLANT18            15
    0     PLANT01            10
    13    PLANT14             1
    

From the above, it seems that Plants 15,17 and 19 are not useful for our orders list so we can eliminate them from the decision variables, while Plant 3 seems dominently satisfying the feasibility of orders (structure and capability wise but capacity is still to be investigated)

Given the above findings, this raises another important question which is: 


    Is there at least one feasible plant for every product in the order list?
    or are there products that cannot be fulfilled at all?


```python
df_feasible_PlantPerProd = mergedProdPlantOrd.groupby('Product_ID').agg({
    'Plant_Code':['count', 'first'],
    'unique_count':['sum']
}).reset_index()
# Flatten the MultiIndex columns and rename them
df_feasible_PlantPerProd.columns = ['_'.join(col).strip() for col in df_feasible_PlantPerProd.columns.values]


df_feasible_PlantPerProd.head()
sorted_feasible_PlantPerProd_df = df_feasible_PlantPerProd.sort_values(by='Plant_Code_count', ascending=True)
print( sorted_feasible_PlantPerProd_df)
```

         Product_ID_  Plant_Code_count Plant_Code_first  unique_count_sum
    0      1613321.0                 1          PLANT03                 5
    507    1692723.0                 1          PLANT16                 4
    508    1692724.0                 1          PLANT16                 2
    509    1692731.0                 1          PLANT16                 1
    510    1692737.0                 1          PLANT16                 2
    ..           ...               ...              ...               ...
    460    1689780.0                 4          PLANT02                16
    453    1689551.0                 4          PLANT02               116
    479    1690616.0                 4          PLANT02               120
    485    1690630.0                 4          PLANT02                96
    346    1683354.0                 5          PLANT05                 5
    
    [772 rows x 4 columns]
    

The above proves that there is at least one supporting plant per product in our orders list. Let's now see the list of plants that HAVE to be utilised i.e. are the only ones to support one or more product in the order list


```python
# Filter rows where column count is equal to 1
sorted_feasible_SSPerProd_df = sorted_feasible_PlantPerProd_df[sorted_feasible_PlantPerProd_df['Plant_Code_count'] == 1]
SSPerProd_df_group = sorted_feasible_SSPerProd_df.groupby('Plant_Code_first')['unique_count_sum'].sum().reset_index()
SSPerProd_df_group.head(20)
# Find unique values in column first
#unique_values = sorted(filtered_df['Planfirst'].unique())

#print("Unique values in column first where column count is 1:")
#print(unique_values)
#print(type(unique_values))
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Plant_Code_first</th>
      <th>unique_count_sum</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>PLANT03</td>
      <td>5769</td>
    </tr>
    <tr>
      <th>1</th>
      <td>PLANT08</td>
      <td>1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>PLANT12</td>
      <td>274</td>
    </tr>
    <tr>
      <th>3</th>
      <td>PLANT13</td>
      <td>71</td>
    </tr>
    <tr>
      <th>4</th>
      <td>PLANT16</td>
      <td>173</td>
    </tr>
  </tbody>
</table>
</div>



The above results in a couple of important findings:
1. Plant 3 HAS to support over half of the orders since it's the only supporting plant to 5769 orders out of 9215  
2. Checking the first Sankey Chart of product-plant possible links above list suggests that we also HAVE to utilise ports 4 and 9 as each of the above plant has a single feasible link to a port [4,4,4,4,9] respectively

### Constraint 4 (Customers-Plant)

This is to investigate all the supported customers/ plants combinations and add a comparative analysis with the customers list of the orders we need to fulfill


```python
df_CustPlant = pd.read_csv('ImperialDACapstone/VmiCustomers.csv')
df_CustPlant.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 9215 entries, 0 to 9214
    Data columns (total 14 columns):
     #   Column       Non-Null Count  Dtype  
    ---  ------       --------------  -----  
     0   Plant Code   14 non-null     object 
     1   Customers    14 non-null     object 
     2   Unnamed: 2   0 non-null      float64
     3   Unnamed: 3   0 non-null      float64
     4   Unnamed: 4   0 non-null      float64
     5   Unnamed: 5   0 non-null      float64
     6   Unnamed: 6   0 non-null      float64
     7   Unnamed: 7   0 non-null      float64
     8   Unnamed: 8   0 non-null      float64
     9   Unnamed: 9   0 non-null      float64
     10  Unnamed: 10  0 non-null      float64
     11  Unnamed: 11  0 non-null      float64
     12  Unnamed: 12  0 non-null      float64
     13  Unnamed: 13  0 non-null      float64
    dtypes: float64(12), object(2)
    memory usage: 1008.0+ KB
    


```python
df_CustPlant.columns = [col.strip().replace(' ', '_').replace('/', '_') for col in df_CustPlant.columns]

df_CustPlant = df_CustPlant[['Plant_Code','Customers']]
df_CustPlant.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Plant_Code</th>
      <th>Customers</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>PLANT02</td>
      <td>V5555555555555_16</td>
    </tr>
    <tr>
      <th>1</th>
      <td>PLANT02</td>
      <td>V555555555555555_29</td>
    </tr>
    <tr>
      <th>2</th>
      <td>PLANT02</td>
      <td>V555555555_3</td>
    </tr>
    <tr>
      <th>3</th>
      <td>PLANT02</td>
      <td>V55555555555555_8</td>
    </tr>
    <tr>
      <th>4</th>
      <td>PLANT02</td>
      <td>V55555555_9</td>
    </tr>
  </tbody>
</table>
</div>



## Investigating the Plants Capability: 
### Plants Capacity Constraint (Constraint 5) and Cost impact

In this section we will study the plants capability and aim to sort them into 4 quadrants of high-low capacity vs high-low cost

| Plants | Ports  |
|--------|--------|
|   a    |     1  |
|   a    |     2  |
|   a    |     3  |
|   b    |     1  |
|   b    |     4  |

Plant a will have a score of 3 (connected to 3 ports) while Plant b has a score of 2. And similarly port 1 has a score of 2 while ports 2,3, and 4 have scores of 1.


```python
dfWHCO = pd.read_csv('ImperialDACapstone/WHCosts.csv')
dfWHCA = pd.read_csv('ImperialDACapstone/Whcapacities.csv')
dfWHCO.columns = [col.strip().replace(' ', '_').replace('/', '_') for col in dfWHCO.columns]
dfWHCA.columns = [col.strip().replace(' ', '_').replace('/', '_') for col in dfWHCA.columns]
WHCOvsCA = dfWHCO.merge(dfWHCA,left_on='WH',right_on='Plant_ID',how='left')
WHCOvsCA['Ratio']=WHCOvsCA['Daily_Capacity']/WHCOvsCA['Cost_unit']
normalized_WHCOvsCA=WHCOvsCA.copy()
normalized_WHCOvsCA=(WHCOvsCA-WHCOvsCA.mean())/WHCOvsCA.std()
WHCOvsCA.head(100)

```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>WH</th>
      <th>Cost_unit</th>
      <th>Plant_ID</th>
      <th>Daily_Capacity</th>
      <th>Ratio</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>PLANT15</td>
      <td>1.42</td>
      <td>PLANT15</td>
      <td>11</td>
      <td>7.746479</td>
    </tr>
    <tr>
      <th>1</th>
      <td>PLANT17</td>
      <td>0.43</td>
      <td>PLANT17</td>
      <td>8</td>
      <td>18.604651</td>
    </tr>
    <tr>
      <th>2</th>
      <td>PLANT18</td>
      <td>2.04</td>
      <td>PLANT18</td>
      <td>111</td>
      <td>54.411765</td>
    </tr>
    <tr>
      <th>3</th>
      <td>PLANT05</td>
      <td>0.49</td>
      <td>PLANT05</td>
      <td>385</td>
      <td>785.714286</td>
    </tr>
    <tr>
      <th>4</th>
      <td>PLANT02</td>
      <td>0.48</td>
      <td>PLANT02</td>
      <td>138</td>
      <td>287.500000</td>
    </tr>
    <tr>
      <th>5</th>
      <td>PLANT01</td>
      <td>0.57</td>
      <td>PLANT01</td>
      <td>1070</td>
      <td>1877.192982</td>
    </tr>
    <tr>
      <th>6</th>
      <td>PLANT06</td>
      <td>0.55</td>
      <td>PLANT06</td>
      <td>49</td>
      <td>89.090909</td>
    </tr>
    <tr>
      <th>7</th>
      <td>PLANT10</td>
      <td>0.49</td>
      <td>PLANT10</td>
      <td>118</td>
      <td>240.816327</td>
    </tr>
    <tr>
      <th>8</th>
      <td>PLANT07</td>
      <td>0.37</td>
      <td>PLANT07</td>
      <td>265</td>
      <td>716.216216</td>
    </tr>
    <tr>
      <th>9</th>
      <td>PLANT14</td>
      <td>0.63</td>
      <td>PLANT14</td>
      <td>549</td>
      <td>871.428571</td>
    </tr>
    <tr>
      <th>10</th>
      <td>PLANT16</td>
      <td>1.92</td>
      <td>PLANT16</td>
      <td>457</td>
      <td>238.020833</td>
    </tr>
    <tr>
      <th>11</th>
      <td>PLANT12</td>
      <td>0.77</td>
      <td>PLANT12</td>
      <td>209</td>
      <td>271.428571</td>
    </tr>
    <tr>
      <th>12</th>
      <td>PLANT11</td>
      <td>0.56</td>
      <td>PLANT11</td>
      <td>332</td>
      <td>592.857143</td>
    </tr>
    <tr>
      <th>13</th>
      <td>PLANT09</td>
      <td>0.47</td>
      <td>PLANT09</td>
      <td>11</td>
      <td>23.404255</td>
    </tr>
    <tr>
      <th>14</th>
      <td>PLANT03</td>
      <td>0.52</td>
      <td>PLANT03</td>
      <td>1013</td>
      <td>1948.076923</td>
    </tr>
    <tr>
      <th>15</th>
      <td>PLANT13</td>
      <td>0.47</td>
      <td>PLANT13</td>
      <td>490</td>
      <td>1042.553191</td>
    </tr>
    <tr>
      <th>16</th>
      <td>PLANT19</td>
      <td>0.64</td>
      <td>PLANT19</td>
      <td>7</td>
      <td>10.937500</td>
    </tr>
    <tr>
      <th>17</th>
      <td>PLANT08</td>
      <td>0.52</td>
      <td>PLANT08</td>
      <td>14</td>
      <td>26.923077</td>
    </tr>
    <tr>
      <th>18</th>
      <td>PLANT04</td>
      <td>0.43</td>
      <td>PLANT04</td>
      <td>554</td>
      <td>1288.372093</td>
    </tr>
  </tbody>
</table>
</div>



## Interesting Finding!
This above table exposes a NEW finding which is the maximum capacity of plant 3. Recall from studying constraint 3, we found out the plant 3 HAS to supply 5769 orders while it's daily capacity is only 1013.
This results in the need to revisit our initial objective function of minimizing cost OR start working on gathering more information/ constraints such as allowed orders backlog window and respective penalty 


```python
import plotly.express as px

# Box plot of daily capacities
fig1 = px.box(dfWHCA, y='Daily_Capacity', title='Distribution of Daily Capacities')
fig1.update_yaxes(title_text='Daily Capacity')

# Scatter plot of cost per unit versus daily capacity
fig2 = px.scatter(WHCOvsCA, x='Cost_unit', y='Daily_Capacity', color='WH', size='Ratio',
                  title='Cost per Unit vs Daily Capacity', labels={'Cost_unit': 'Cost per Unit', 'Daily_Capacity': 'Daily Capacity'})
# Slit graph into 4 quadrants
mean_x = WHCOvsCA['Cost_unit'].mean()
mean_y = WHCOvsCA['Daily_Capacity'].mean()
fig2.add_shape(type="line",
              x0=mean_x, y0=WHCOvsCA['Daily_Capacity'].min(),
              x1=mean_x, y1=WHCOvsCA['Daily_Capacity'].max(),
              line=dict(color="Red", width=2, dash="dash"))
fig2.add_shape(type="line",
              x0=WHCOvsCA['Cost_unit'].min(), y0=mean_y,
              x1=WHCOvsCA['Cost_unit'].max(), y1=mean_y,
              line=dict(color="Red", width=2, dash="dash"))
fig2.add_annotation(x=mean_x, y=mean_y, text="Mean", showarrow=False, yshift=10, xshift=10)
fig2.update_layout(
    shapes=[
        # Horizontal line
        dict(
            type="line",
            x0=mean_x, y0=WHCOvsCA['Daily_Capacity'].min(),
            x1=mean_x, y1=WHCOvsCA['Daily_Capacity'].max(),
            line=dict(color="Red", width=2, dash="dash")
        ),
        # Vertical line
        dict(
            type="line",
            x0=WHCOvsCA['Cost_unit'].min(), y0=mean_y,
            x1=WHCOvsCA['Cost_unit'].max(), y1=mean_y,
            line=dict(color="Red", width=2, dash="dash")
        )
    ]
)


# Display the plots
fig1.show()
fig2.show()
```


<div>                            <div id="5fd35509-c64a-4a05-8e84-3d8199e68302" class="plotly-graph-div" style="height:525px; width:100%;"></div>            <script type="text/javascript">                require(["plotly"], function(Plotly) {                    window.PLOTLYENV=window.PLOTLYENV || {};                                    if (document.getElementById("5fd35509-c64a-4a05-8e84-3d8199e68302")) {                    Plotly.newPlot(                        "5fd35509-c64a-4a05-8e84-3d8199e68302",                        [{"alignmentgroup":"True","hovertemplate":"Daily_Capacity=%{y}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"","marker":{"color":"#636efa"},"name":"","notched":false,"offsetgroup":"","orientation":"v","showlegend":false,"x0":" ","xaxis":"x","y":[11,8,111,385,138,1070,49,118,265,549,457,209,332,11,1013,490,7,14,554],"y0":" ","yaxis":"y","type":"box"}],                        {"template":{"data":{"histogram2dcontour":[{"type":"histogram2dcontour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"choropleth":[{"type":"choropleth","colorbar":{"outlinewidth":0,"ticks":""}}],"histogram2d":[{"type":"histogram2d","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmap":[{"type":"heatmap","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmapgl":[{"type":"heatmapgl","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"contourcarpet":[{"type":"contourcarpet","colorbar":{"outlinewidth":0,"ticks":""}}],"contour":[{"type":"contour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"surface":[{"type":"surface","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"mesh3d":[{"type":"mesh3d","colorbar":{"outlinewidth":0,"ticks":""}}],"scatter":[{"fillpattern":{"fillmode":"overlay","size":10,"solidity":0.2},"type":"scatter"}],"parcoords":[{"type":"parcoords","line":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolargl":[{"type":"scatterpolargl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"bar":[{"error_x":{"color":"#2a3f5f"},"error_y":{"color":"#2a3f5f"},"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"bar"}],"scattergeo":[{"type":"scattergeo","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolar":[{"type":"scatterpolar","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"histogram":[{"marker":{"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"histogram"}],"scattergl":[{"type":"scattergl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatter3d":[{"type":"scatter3d","line":{"colorbar":{"outlinewidth":0,"ticks":""}},"marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattermapbox":[{"type":"scattermapbox","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterternary":[{"type":"scatterternary","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattercarpet":[{"type":"scattercarpet","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"carpet":[{"aaxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"baxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"type":"carpet"}],"table":[{"cells":{"fill":{"color":"#EBF0F8"},"line":{"color":"white"}},"header":{"fill":{"color":"#C8D4E3"},"line":{"color":"white"}},"type":"table"}],"barpolar":[{"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"barpolar"}],"pie":[{"automargin":true,"type":"pie"}]},"layout":{"autotypenumbers":"strict","colorway":["#636efa","#EF553B","#00cc96","#ab63fa","#FFA15A","#19d3f3","#FF6692","#B6E880","#FF97FF","#FECB52"],"font":{"color":"#2a3f5f"},"hovermode":"closest","hoverlabel":{"align":"left"},"paper_bgcolor":"white","plot_bgcolor":"#E5ECF6","polar":{"bgcolor":"#E5ECF6","angularaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"radialaxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"ternary":{"bgcolor":"#E5ECF6","aaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"baxis":{"gridcolor":"white","linecolor":"white","ticks":""},"caxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"coloraxis":{"colorbar":{"outlinewidth":0,"ticks":""}},"colorscale":{"sequential":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"sequentialminus":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"diverging":[[0,"#8e0152"],[0.1,"#c51b7d"],[0.2,"#de77ae"],[0.3,"#f1b6da"],[0.4,"#fde0ef"],[0.5,"#f7f7f7"],[0.6,"#e6f5d0"],[0.7,"#b8e186"],[0.8,"#7fbc41"],[0.9,"#4d9221"],[1,"#276419"]]},"xaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"yaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"scene":{"xaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"yaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"zaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2}},"shapedefaults":{"line":{"color":"#2a3f5f"}},"annotationdefaults":{"arrowcolor":"#2a3f5f","arrowhead":0,"arrowwidth":1},"geo":{"bgcolor":"white","landcolor":"#E5ECF6","subunitcolor":"white","showland":true,"showlakes":true,"lakecolor":"white"},"title":{"x":0.05},"mapbox":{"style":"light"}}},"xaxis":{"anchor":"y","domain":[0.0,1.0]},"yaxis":{"anchor":"x","domain":[0.0,1.0],"title":{"text":"Daily Capacity"}},"legend":{"tracegroupgap":0},"title":{"text":"Distribution of Daily Capacities"},"boxmode":"group"},                        {"responsive": true}                    ).then(function(){

var gd = document.getElementById('5fd35509-c64a-4a05-8e84-3d8199e68302');
var x = new MutationObserver(function (mutations, observer) {{
        var display = window.getComputedStyle(gd).display;
        if (!display || display === 'none') {{
            console.log([gd, 'removed!']);
            Plotly.purge(gd);
            observer.disconnect();
        }}
}});

// Listen for the removal of the full notebook cells
var notebookContainer = gd.closest('#notebook-container');
if (notebookContainer) {{
    x.observe(notebookContainer, {childList: true});
}}

// Listen for the clearing of the current output cell
var outputEl = gd.closest('.output');
if (outputEl) {{
    x.observe(outputEl, {childList: true});
}}

                        })                };                });            </script>        </div>



<div>                            <div id="d21bb787-40a0-4b42-89cd-6a12f412775a" class="plotly-graph-div" style="height:525px; width:100%;"></div>            <script type="text/javascript">                require(["plotly"], function(Plotly) {                    window.PLOTLYENV=window.PLOTLYENV || {};                                    if (document.getElementById("d21bb787-40a0-4b42-89cd-6a12f412775a")) {                    Plotly.newPlot(                        "d21bb787-40a0-4b42-89cd-6a12f412775a",                        [{"hovertemplate":"WH=PLANT15\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT15","marker":{"color":"#636efa","size":[7.746478873239437],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT15","orientation":"v","showlegend":true,"x":[1.42],"xaxis":"x","y":[11],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT17\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT17","marker":{"color":"#EF553B","size":[18.6046511627907],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT17","orientation":"v","showlegend":true,"x":[0.43],"xaxis":"x","y":[8],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT18\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT18","marker":{"color":"#00cc96","size":[54.411764705882355],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT18","orientation":"v","showlegend":true,"x":[2.04],"xaxis":"x","y":[111],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT05\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT05","marker":{"color":"#ab63fa","size":[785.7142857142858],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT05","orientation":"v","showlegend":true,"x":[0.49],"xaxis":"x","y":[385],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT02\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT02","marker":{"color":"#FFA15A","size":[287.5],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT02","orientation":"v","showlegend":true,"x":[0.48],"xaxis":"x","y":[138],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT01\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT01","marker":{"color":"#19d3f3","size":[1877.1929824561405],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT01","orientation":"v","showlegend":true,"x":[0.57],"xaxis":"x","y":[1070],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT06\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT06","marker":{"color":"#FF6692","size":[89.09090909090908],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT06","orientation":"v","showlegend":true,"x":[0.55],"xaxis":"x","y":[49],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT10\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT10","marker":{"color":"#B6E880","size":[240.81632653061226],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT10","orientation":"v","showlegend":true,"x":[0.49],"xaxis":"x","y":[118],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT07\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT07","marker":{"color":"#FF97FF","size":[716.2162162162163],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT07","orientation":"v","showlegend":true,"x":[0.37],"xaxis":"x","y":[265],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT14\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT14","marker":{"color":"#FECB52","size":[871.4285714285714],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT14","orientation":"v","showlegend":true,"x":[0.63],"xaxis":"x","y":[549],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT16\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT16","marker":{"color":"#636efa","size":[238.02083333333334],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT16","orientation":"v","showlegend":true,"x":[1.92],"xaxis":"x","y":[457],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT12\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT12","marker":{"color":"#EF553B","size":[271.42857142857144],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT12","orientation":"v","showlegend":true,"x":[0.77],"xaxis":"x","y":[209],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT11\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT11","marker":{"color":"#00cc96","size":[592.8571428571428],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT11","orientation":"v","showlegend":true,"x":[0.56],"xaxis":"x","y":[332],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT09\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT09","marker":{"color":"#ab63fa","size":[23.404255319148938],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT09","orientation":"v","showlegend":true,"x":[0.47],"xaxis":"x","y":[11],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT03\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT03","marker":{"color":"#FFA15A","size":[1948.076923076923],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT03","orientation":"v","showlegend":true,"x":[0.52],"xaxis":"x","y":[1013],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT13\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT13","marker":{"color":"#19d3f3","size":[1042.5531914893618],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT13","orientation":"v","showlegend":true,"x":[0.47],"xaxis":"x","y":[490],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT19\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT19","marker":{"color":"#FF6692","size":[10.9375],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT19","orientation":"v","showlegend":true,"x":[0.64],"xaxis":"x","y":[7],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT08\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT08","marker":{"color":"#B6E880","size":[26.923076923076923],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT08","orientation":"v","showlegend":true,"x":[0.52],"xaxis":"x","y":[14],"yaxis":"y","type":"scatter"},{"hovertemplate":"WH=PLANT04\u003cbr\u003eCost per Unit=%{x}\u003cbr\u003eDaily Capacity=%{y}\u003cbr\u003eRatio=%{marker.size}\u003cextra\u003e\u003c\u002fextra\u003e","legendgroup":"PLANT04","marker":{"color":"#FF97FF","size":[1288.3720930232557],"sizemode":"area","sizeref":4.8701923076923075,"symbol":"circle"},"mode":"markers","name":"PLANT04","orientation":"v","showlegend":true,"x":[0.43],"xaxis":"x","y":[554],"yaxis":"y","type":"scatter"}],                        {"template":{"data":{"histogram2dcontour":[{"type":"histogram2dcontour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"choropleth":[{"type":"choropleth","colorbar":{"outlinewidth":0,"ticks":""}}],"histogram2d":[{"type":"histogram2d","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmap":[{"type":"heatmap","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"heatmapgl":[{"type":"heatmapgl","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"contourcarpet":[{"type":"contourcarpet","colorbar":{"outlinewidth":0,"ticks":""}}],"contour":[{"type":"contour","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"surface":[{"type":"surface","colorbar":{"outlinewidth":0,"ticks":""},"colorscale":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]]}],"mesh3d":[{"type":"mesh3d","colorbar":{"outlinewidth":0,"ticks":""}}],"scatter":[{"fillpattern":{"fillmode":"overlay","size":10,"solidity":0.2},"type":"scatter"}],"parcoords":[{"type":"parcoords","line":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolargl":[{"type":"scatterpolargl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"bar":[{"error_x":{"color":"#2a3f5f"},"error_y":{"color":"#2a3f5f"},"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"bar"}],"scattergeo":[{"type":"scattergeo","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterpolar":[{"type":"scatterpolar","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"histogram":[{"marker":{"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"histogram"}],"scattergl":[{"type":"scattergl","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatter3d":[{"type":"scatter3d","line":{"colorbar":{"outlinewidth":0,"ticks":""}},"marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattermapbox":[{"type":"scattermapbox","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scatterternary":[{"type":"scatterternary","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"scattercarpet":[{"type":"scattercarpet","marker":{"colorbar":{"outlinewidth":0,"ticks":""}}}],"carpet":[{"aaxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"baxis":{"endlinecolor":"#2a3f5f","gridcolor":"white","linecolor":"white","minorgridcolor":"white","startlinecolor":"#2a3f5f"},"type":"carpet"}],"table":[{"cells":{"fill":{"color":"#EBF0F8"},"line":{"color":"white"}},"header":{"fill":{"color":"#C8D4E3"},"line":{"color":"white"}},"type":"table"}],"barpolar":[{"marker":{"line":{"color":"#E5ECF6","width":0.5},"pattern":{"fillmode":"overlay","size":10,"solidity":0.2}},"type":"barpolar"}],"pie":[{"automargin":true,"type":"pie"}]},"layout":{"autotypenumbers":"strict","colorway":["#636efa","#EF553B","#00cc96","#ab63fa","#FFA15A","#19d3f3","#FF6692","#B6E880","#FF97FF","#FECB52"],"font":{"color":"#2a3f5f"},"hovermode":"closest","hoverlabel":{"align":"left"},"paper_bgcolor":"white","plot_bgcolor":"#E5ECF6","polar":{"bgcolor":"#E5ECF6","angularaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"radialaxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"ternary":{"bgcolor":"#E5ECF6","aaxis":{"gridcolor":"white","linecolor":"white","ticks":""},"baxis":{"gridcolor":"white","linecolor":"white","ticks":""},"caxis":{"gridcolor":"white","linecolor":"white","ticks":""}},"coloraxis":{"colorbar":{"outlinewidth":0,"ticks":""}},"colorscale":{"sequential":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"sequentialminus":[[0.0,"#0d0887"],[0.1111111111111111,"#46039f"],[0.2222222222222222,"#7201a8"],[0.3333333333333333,"#9c179e"],[0.4444444444444444,"#bd3786"],[0.5555555555555556,"#d8576b"],[0.6666666666666666,"#ed7953"],[0.7777777777777778,"#fb9f3a"],[0.8888888888888888,"#fdca26"],[1.0,"#f0f921"]],"diverging":[[0,"#8e0152"],[0.1,"#c51b7d"],[0.2,"#de77ae"],[0.3,"#f1b6da"],[0.4,"#fde0ef"],[0.5,"#f7f7f7"],[0.6,"#e6f5d0"],[0.7,"#b8e186"],[0.8,"#7fbc41"],[0.9,"#4d9221"],[1,"#276419"]]},"xaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"yaxis":{"gridcolor":"white","linecolor":"white","ticks":"","title":{"standoff":15},"zerolinecolor":"white","automargin":true,"zerolinewidth":2},"scene":{"xaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"yaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2},"zaxis":{"backgroundcolor":"#E5ECF6","gridcolor":"white","linecolor":"white","showbackground":true,"ticks":"","zerolinecolor":"white","gridwidth":2}},"shapedefaults":{"line":{"color":"#2a3f5f"}},"annotationdefaults":{"arrowcolor":"#2a3f5f","arrowhead":0,"arrowwidth":1},"geo":{"bgcolor":"white","landcolor":"#E5ECF6","subunitcolor":"white","showland":true,"showlakes":true,"lakecolor":"white"},"title":{"x":0.05},"mapbox":{"style":"light"}}},"xaxis":{"anchor":"y","domain":[0.0,1.0],"title":{"text":"Cost per Unit"}},"yaxis":{"anchor":"x","domain":[0.0,1.0],"title":{"text":"Daily Capacity"}},"legend":{"title":{"text":"WH"},"tracegroupgap":0,"itemsizing":"constant"},"title":{"text":"Cost per Unit vs Daily Capacity"},"shapes":[{"line":{"color":"Red","dash":"dash","width":2},"type":"line","x0":0.7247368421052632,"x1":0.7247368421052632,"y0":7,"y1":1070},{"line":{"color":"Red","dash":"dash","width":2},"type":"line","x0":0.37,"x1":2.04,"y0":304.7894736842105,"y1":304.7894736842105}],"annotations":[{"showarrow":false,"text":"Mean","x":0.7247368421052632,"xshift":10,"y":304.7894736842105,"yshift":10}]},                        {"responsive": true}                    ).then(function(){

var gd = document.getElementById('d21bb787-40a0-4b42-89cd-6a12f412775a');
var x = new MutationObserver(function (mutations, observer) {{
        var display = window.getComputedStyle(gd).display;
        if (!display || display === 'none') {{
            console.log([gd, 'removed!']);
            Plotly.purge(gd);
            observer.disconnect();
        }}
}});

// Listen for the removal of the full notebook cells
var notebookContainer = gd.closest('#notebook-container');
if (notebookContainer) {{
    x.observe(notebookContainer, {childList: true});
}}

// Listen for the clearing of the current output cell
var outputEl = gd.closest('.output');
if (outputEl) {{
    x.observe(outputEl, {childList: true});
}}

                        })                };                });            </script>        </div>



```python
!jupyter nbconvert --to markdown ScLogisticsOpt-Copy1.ipynb
```

    [NbConvertApp] Converting notebook ScLogisticsOpt-Copy1.ipynb to markdown
    [NbConvertApp] Writing 3731180 bytes to ScLogisticsOpt-Copy1.md
    

