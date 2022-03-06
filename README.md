# AlzheimersGeneNetwork
import pandas as pd
import json
import os
import dash
from dash.dependencies import Input, Output, State
from dash import dcc
from dash import html
import dash_cytoscape as cyto
from demos import dash_reusable_components as drc

app = dash.Dash(__name__)
server = app.server
app.title = "Alzheimer\'s Gene Network"
i_edge_data_file = '/Users/juliawarnken/Documents/UConn Grad School/CSE5099 Independent Study/finaldash/data/new_edges2.csv'
i_alz_gene_vals_file = '/Users/juliawarnken/Documents/UConn Grad School/CSE5099 Independent Study/finaldash/data/GSE44768_CR_alz_avg_gene_vals.csv'
i_nd_gene_vals_file = '/Users/juliawarnken/Documents/UConn Grad School/CSE5099 Independent Study/finaldash/data/GSE44768_CR_nd_avg_gene_vals.csv'

# ###################### DATA PREPROCESSING ######################
# Load data
edges = pd.read_csv(i_edge_data_file)
nodes = set()

following_node_di = {}  # user id -> list of users they are following
following_edges_di = {}  # user id -> list of cy edges starting from user id

followers_node_di = {}  # user id -> list of followers (cy_node format)
followers_edges_di = {}  # user id -> list of cy edges ending at user id

cy_edges = []
cy_nodes = []

alz_node_sizes = pd.read_csv(i_alz_gene_vals_file)
nd_node_sizes = pd.read_csv(i_nd_gene_vals_file)
alz_nodes = alz_node_sizes['Gene']
nd_nodes = nd_node_sizes['Gene']
for index, row in edges.iterrows():
    source, target, weight = row['node1'], row['node2'], row['weight']
    cy_edge = {'data': {'id': source + target, 'source': source, 'target': target,
                        'weight': weight, 'rounded_weight': round(weight, 3)}, 'classes': 'off'}

    nodes.add(source)
    for g in range(len(alz_nodes)):
        if str(alz_node_sizes.loc[g, 'Gene']) == str(source):
            alz_avg = alz_node_sizes.loc[g, 'Average']
            nd_avg = nd_node_sizes.loc[g, 'Average']
            cy_source = {"data": {"id": source, "label": source, "Alz Average": alz_avg, "ND Average": nd_avg,
                                  "Rounded Alz Average": round(alz_avg, 3), "Rounded ND Average": round(nd_avg, 3)},
                        'classes': 'off'}
            cy_nodes.append(cy_source)

    nodes.add(target)
    for g in range(len(nd_nodes)):
        if str(nd_node_sizes.loc[g, 'Gene']) == str(target):
            alz_avg = alz_node_sizes.loc[g, 'Average']
            nd_avg = nd_node_sizes.loc[g, 'Average']
            cy_target = {"data": {"id": target, "label": target, "Alz Average": alz_avg, "ND Average": nd_avg,
                                  "Rounded Alz Average": round(alz_avg, 3), "Rounded ND Average": round(nd_avg, 3)},
                         'classes': 'off'}
            cy_nodes.append(cy_target)

    # Process dictionary of following
    if not following_node_di.get(source):
        following_node_di[source] = []
    if not following_edges_di.get(source):
        following_edges_di[source] = []

    following_node_di[source].append(cy_target)
    following_edges_di[source].append(cy_edge)

    # Process dictionary of followers
    if not followers_node_di.get(target):
        followers_node_di[target] = []
    if not followers_edges_di.get(target):
        followers_edges_di[target] = []

    followers_node_di[target].append(cy_source)
    followers_edges_di[target].append(cy_edge)

    cy_edges.append(cy_edge)

default_stylesheet = [
    {"selector": 'node',
     'style': {"opacity": 1, "label": "data(label)", "text-opacity": 1, "font-size": 12, 'z-index': 9999}},
    # 'selectable': True, 'selected': False}},
    {"selector": 'edge',
     'style': {"curve-style": "bezier", "opacity": 1}}

]

styles = {
    'json-output': {'overflow-y': 'scroll', 'height': 'calc(50% - 25px)', 'border': 'thin lightgrey solid'},
    'tab': {'height': 'calc(98vh - 105px)'}}

SIDEBAR_STYLE = {
    'position': 'fixed',
    'top': 0,
    'left': '5px',
    'bottom': 0,
    'width': '20%',
    # 'margin-left': '5px',
    # 'padding': '20px 10px',
    'background-color': '#f8f9fa'
}

CONTENT_STYLE = {
    'top': 0,
    'left': 0,
    'bottom': 0,
    # 'width': '80%',
    'margin-left': '25%',
    'margin-right': '5%',
    'max-height': '90%'
}
app.layout = html.Div([
    html.Div(className='five columns', children=[
        dcc.Tabs(id='tabs', children=[
            dcc.Tab(label='Control Panel', children=[
                drc.NamedDropdown(name='Layout', id='dropdown-layout', value='circle', clearable=False,
                                  options=drc.DropdownOptionsList('random', 'grid', 'circle',
                                                                  'concentric', 'breadthfirst', 'cose'), ),

                drc.NamedDropdown(name='Node Shape', id='dropdown-node-shape', value='ellipse', clearable=False,
                                  options=drc.DropdownOptionsList('ellipse', 'triangle', 'rectangle', 'diamond',
                                                                  'pentagon',
                                                                  'hexagon', 'heptagon', 'octagon', 'star',
                                                                  'polygon', )),

                drc.NamedInput(
                    name='Followers Color', id='input-follower-color', type='text', value='skyblue'),

                drc.NamedInput(
                    name='Following Color', id='input-following-color', type='text', value='darkblue'),
                html.P(id='cytoscape-mouseoverNodeData-output'),
                html.P(id='cytoscape-mouseoverEdgeData-output'),
                dcc.Markdown(id='cytoscape-selectedNodeData-markdown'),
                dcc.RadioItems(id='hops',
                options=[{'label': i, 'value': i} for i in ['H1', 'H2', 'H3']],
                value='H1',
                labelStyle={'display': 'inline-block'}
            )
            ]),

            dcc.Tab(label='JSON', children=[
                html.Div(style=styles['tab'], children=[
                    html.P('Node Object JSON:'), html.Pre(
                        id='tap-node-json-output', style=styles['json-output']),
                    html.P('Edge Object JSON:'), html.Pre(id='tap-edge-json-output', style=styles['json-output'])])
            ])
        ]),
    ],
             style=SIDEBAR_STYLE, ),
        html.Div([
        html.Div([html.H1("Alzheimer\'s Gene Network Graph")],
                 className="row", style={'textAlign': "center"}),
        html.Div(className='eight columns', children=[
            cyto.Cytoscape(id='cytoscape',
                           elements=cy_edges + cy_nodes,
                           style={'height': '85vh', 'width': '100%'}, responsive=True)]),

    ], style=CONTENT_STYLE)

])


@app.callback(Output('cytoscape-mouseoverNodeData-output', 'children'),
              Input('cytoscape', 'mouseoverNodeData'))
def displayTapNodeData(data):
    if not data:
        return None
    if data:
        return "You recently hovered over the gene " + data['label'] + ". Alz Average: " + str(
            data['Rounded Alz Average']) + "; ND Average: " + str(data['Rounded ND Average'])


@app.callback(Output('cytoscape-mouseoverEdgeData-output', 'children'),
              Input('cytoscape', 'mouseoverEdgeData'))
def displayTapEdgeData(data):
    if not data:
        return None
    if data:
        return "You recently hovered over the edge from " + data['source'].upper() + " to " + data[
            'target'].upper() + ". Correlation: " + str(data['rounded_weight'])


@app.callback(Output('cytoscape-selectedNodeData-markdown', 'children'),
              Input('cytoscape', 'selectedNodeData'))
def displaySelectedNodeData(data_list):
    if not data_list:  # is None:
        return "No gene selected."
    genes_list = [data['label'] for data in data_list]
    if len(genes_list) == 1:
        return "You selected the gene: " + "\n* ".join(genes_list)
    if len(genes_list) > 1:
        return "You selected the genes: " + "\n* ".join(genes_list)


@app.callback(Output('tap-node-json-output', 'children'),
              [Input('cytoscape', 'tapNode')])
def display_tap_node(data):
    return json.dumps(data, indent=2)


@app.callback(Output('tap-edge-json-output', 'children'),
              [Input('cytoscape', 'tapEdge')])
def display_tap_edge(data):
    return json.dumps(data, indent=2)


@app.callback(Output('cytoscape', 'layout'),
              [Input('dropdown-layout', 'value')])
def update_cytoscape_layout(layout):
    return {'name': layout}


@app.callback(
    [Output('cytoscape', 'elements'),
     Output('cytoscape', 'stylesheet')],
    [Input('cytoscape', 'selectedNodeData'),
     Input('input-follower-color', 'value'),
     Input('input-following-color', 'value'),
     Input('dropdown-node-shape', 'value'),
     State('cytoscape', 'elements'),
     Input('hops', 'value')]
)
def onClickNode(selected_nodes_data, follower_color, following_color, node_shape, elements, hops):
    selected_node_ids = []
    if selected_nodes_data:
        selected_node_ids = [item['id'] for item in selected_nodes_data]
    element_count = len(elements)

    following_nodes = []
    follower_nodes = []
    pos_edges = []
    neg_edges = []
    for i in range(element_count):
        item = elements[i]
        if 'source' in item['data']:
            if item['data']['source'] in selected_node_ids:
                # item['data']['selected'] = "True"
                follower_nodes.append(item['data']['target'])
                if item['data']['weight'] > 0:
                    pos_edges.append(item['data']['id'])
                if item['data']['weight'] < 0:
                    neg_edges.append(item['data']['id'])
            elif item['data']['target'] in selected_node_ids:
                # item['data']['selected'] = "True"
                following_nodes.append(item['data']['source'])
                if item['data']['weight'] > 0:
                    pos_edges.append(item['data']['id'])
                if item['data']['weight'] < 0:
                    neg_edges.append(item['data']['id'])
            else:
                item['data']['selected'] = "False"

    following2 = []
    follower2 = []
    pos2 = []
    neg2 = []
    following3 = []
    follower3 = []
    pos3 = []
    neg3 = []
    if len(selected_node_ids) == 1:
        if hops == 'H2':
            for i in range(element_count):
                item = elements[i]
                if 'source' in item['data']:
                    if item['data']['source'] in following_nodes:
                        following2.append(item['data']['source'])
                        follower2.append(item['data']['target'])
                        if item['data']['weight'] > 0:
                            pos2.append(item['data']['id'])
                        if item['data']['weight'] < 0:
                            neg2.append(item['data']['id'])
                    elif item['data']['target'] in following_nodes:
                        following2.append(item['data']['target'])
                        follower2.append(item['data']['source'])
                        if item['data']['weight'] > 0:
                            pos2.append(item['data']['id'])
                        if item['data']['weight'] < 0:
                            neg2.append(item['data']['id'])
                    elif item['data']['source'] in follower_nodes:
                        follower2.append(item['data']['source'])
                        following2.append(item['data']['target'])
                        if item['data']['weight'] > 0:
                            pos2.append(item['data']['id'])
                        if item['data']['weight'] < 0:
                            neg2.append(item['data']['id'])
                    elif item['data']['target'] in follower_nodes:
                        follower2.append(item['data']['target'])
                        following2.append(item['data']['source'])
                        if item['data']['weight'] > 0:
                            pos2.append(item['data']['id'])
                        if item['data']['weight'] < 0:
                            neg2.append(item['data']['id'])
                    else:
                        item['data']['selected'] = "False"

        if hops == 'H3':
            for i in range(element_count):
                item = elements[i]
                if 'source' in item['data']:
                    if item['data']['source'] in following_nodes:
                        following2.append(item['data']['source'])
                        follower2.append(item['data']['target'])
                        if item['data']['weight'] > 0:
                            pos2.append(item['data']['id'])
                        if item['data']['weight'] < 0:
                            neg2.append(item['data']['id'])
                    elif item['data']['target'] in following_nodes:
                        following2.append(item['data']['target'])
                        follower2.append(item['data']['source'])
                        if item['data']['weight'] > 0:
                            pos2.append(item['data']['id'])
                        if item['data']['weight'] < 0:
                            neg2.append(item['data']['id'])
                    elif item['data']['source'] in follower_nodes:
                        follower2.append(item['data']['source'])
                        following2.append(item['data']['target'])
                        if item['data']['weight'] > 0:
                            pos2.append(item['data']['id'])
                        if item['data']['weight'] < 0:
                            neg2.append(item['data']['id'])
                    elif item['data']['target'] in follower_nodes:
                        follower2.append(item['data']['target'])
                        following2.append(item['data']['source'])
                        if item['data']['weight'] > 0:
                            pos2.append(item['data']['id'])
                        if item['data']['weight'] < 0:
                            neg2.append(item['data']['id'])
                    elif item['data']['source'] in following2:
                        following3.append(item['data']['source'])
                        follower3.append(item['data']['target'])
                        if item['data']['weight'] > 0:
                            pos3.append(item['data']['id'])
                        if item['data']['weight'] < 0:
                            neg3.append(item['data']['id'])
                    elif item['data']['target'] in following2:
                        following3.append(item['data']['target'])
                        follower3.append(item['data']['source'])
                        if item['data']['weight'] > 0:
                            pos3.append(item['data']['id'])
                        if item['data']['weight'] < 0:
                            neg3.append(item['data']['id'])
                    elif item['data']['source'] in follower2:
                        follower3.append(item['data']['source'])
                        following3.append(item['data']['target'])
                        if item['data']['weight'] > 0:
                            pos2.append(item['data']['id'])
                        if item['data']['weight'] < 0:
                            neg3.append(item['data']['id'])
                    elif item['data']['target'] in follower2:
                        follower3.append(item['data']['target'])
                        following3.append(item['data']['source'])
                        if item['data']['weight'] > 0:
                            pos3.append(item['data']['id'])
                        if item['data']['weight'] < 0:
                            neg3.append(item['data']['id'])
                    else:
                        item['data']['selected'] = "False"


    for item in elements:
        item['data']['selected'] = "False"
        if item['data']['id'] in selected_node_ids:
            item['data']['selected'] = "selected"
        elif item['data']['id'] in follower_nodes:
            item['data']['selected'] = "followerNode"
        elif item['data']['id'] in following_nodes:
            item['data']['selected'] = "followingNode"
        elif item['data']['id'] in pos_edges:
            item['data']['selected'] = 'pos'
        elif item['data']['id'] in neg_edges:
            item['data']['selected'] = 'neg'
        elif item['data']['id'] in follower2:
            item['data']['selected'] = "followerNode2"
        elif item['data']['id'] in following2:
            item['data']['selected'] = "followingNode2"
        elif item['data']['id'] in pos2:
            item['data']['selected'] = 'pos'
        elif item['data']['id'] in neg2:
            item['data']['selected'] = 'neg'
        elif item['data']['id'] in follower3:
            item['data']['selected'] = "followerNode3"
        elif item['data']['id'] in following3:
            item['data']['selected'] = "followingNode3"
        elif item['data']['id'] in pos3:
            item['data']['selected'] = 'pos'
        elif item['data']['id'] in neg3:
            item['data']['selected'] = 'neg'

    stylesheet = [
        {
            "selector": 'node',
            'style': {
                'opacity': 1,
                'shape': node_shape,
                "label": "data(label)",
                "text-opacity": 1,
                "font-size": 12,
                'z-index': 9999
            }
        },
        {
            'selector': 'edge',
            'style': {
                'opacity': 1,
                "curve-style": "bezier"
            }
        },
        {
            "selector": ':selected',
            "style": {
                'background-color': 'blue', "opacity": 1,
                "border-color": "blue", "border-width": 3, "border-opacity": 1,
                "label": "data(label)", "color": "blue", "text-opacity": 1, "font-size": 14, 'z-index': 9999
            }
        },
        {
            "selector": '[selected = "followingNode"]',
            "style": {
                'background-color': following_color, "opacity": 1,
                "label": "data(label)", "color": following_color, "text-opacity": 1, "font-size": 14
            }
        },
        {
            "selector": '[selected = "followerNode"]',
            "style": {
                'background-color': follower_color, "opacity": 1,
                "label": "data(label)", "color": follower_color, "text-opacity": 1, "font-size": 14
            }
        },
        {
            "selector": '[selected = "followingNode2"]',
            "style": {
                'background-color': "darkviolet", "opacity": 1,
                "label": "data(label)", "color": "darkviolet", "text-opacity": 1, "font-size": 14
            }
        },
        {
            "selector": '[selected = "followerNode2"]',
            "style": {
                'background-color': "darkviolet", "opacity": 1,
                "label": "data(label)", "color": "darkviolet", "text-opacity": 1, "font-size": 14
            }
        },
        {
            "selector": '[selected = "followingNode3"]',
            "style": {
                'background-color': "hotpink", "opacity": 1,
                "label": "data(label)", "color": "hotpink", "text-opacity": 1, "font-size": 14
            }
        },
        {
            "selector": '[selected = "followerNode3"]',
            "style": {
                'background-color': "hotpink", "opacity": 1,
                "label": "data(label)", "color": "hotpink", "text-opacity": 1, "font-size": 14
            }
        },
        {
            "selector": '[selected = "pos"]',
            "style": {"target-arrow-color": 'red',
                      "target-arrow-shape": "triangle",
                      "line-color": 'red', "label": "data(rounded_weight)",
                      "arrow-scale": 2,
                      'opacity': 1
                      }
        },
        {
            "selector": '[selected = "neg"]',
            "style": {"target-arrow-color": 'green',
                      "target-arrow-shape": "tee",
                      "line-color": 'green', "label": "data(rounded_weight)",
                      "arrow-scale": 2,
                      'opacity': 1
                      }
        }
    ]

    return [elements, stylesheet]


if __name__ == '__main__':
    app.run_server(debug=True, host='127.0.0.1', port=8050)
