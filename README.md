# weather-model-graphs

## Installation

```
pdm install
```

### pytorch support

cpu only:

```bash
PIP_INDEX_URL=https://download.pytorch.org/whl/cpu pdm install --group pytorch
```

gpu support (see https://pytorch.org/get-started/locally/#linux-pip for older versions of CUDA):


```bash
pdm install --group pytorch
```

# Background and design

The only input the graph generation in `weather-model-graphs` requires is the static `(x,y)` *grid* coordinates of the atmospheric state as the state changes over time. These coordinates are used to create the **grid nodes** nodes of the graph, with a node for each `(x,y)` coordinate. In addition the graph contains **mesh nodes** that represent the latent space of the model at a set of `(x,y)` coordinates (this are in general a different set of coordinates to the **grid nodes** coordinates).

In summary, there are two sets of nodes in the graph:

- **grid nodes**: represent the physical variables of the atmospheric state at a specific `(x,y)` coordinate

- **mesh nodes**: represent the latent space of the model at specific `(x,y)` coordinate

With these two sets of nodes, the graph is constructed by connecting the **grid nodes** to the **mesh nodes** and the **mesh nodes** to each other.
The edges between the **grid nodes** and the **mesh nodes** represent the encoding of the physical variables into the latent space of the model, while the edges between the **mesh nodes** represent the processing of the latent space through the time evolution of the atmospheric state.

In summary, the complete message-passing graph consists of three components:

- **grid-to-mesh** (`g2m`): the encoding compenent, where edges represent the encoding of physical variables into the latent space of the model

- **mesh-to-mesh** (`m2m`): the processing component, where edges represent information flow between nodes updating the latent presentation at mesh nodes through the time evolution of the atmospheric state

- **mesh-to-grid** (`m2g`): the decoding component, where edges represent the decoding of the latent space back into physical variables


## Design principles


The graph generation in `weather-model-graphs` is split into to the following steps:

1. Create the three graph components of the message-passing graph that constitute the auto-regressive atmospheric flow model, all represented by `networkx.DiGraph` objects:

    - **grid-to-mesh** (`g2m`): the encoding compenent, where edges represent the encoding of physical variables into the latent space of the model

    - **mesh-to-mesh** (`m2m`): the processing component, where edges represent information flow between nodes through the time evolution of the atmospheric state

    - **mesh-to-grid** (`m2g`): the decoding component, where edges represent the decoding of the latent space back into physical variables

2. Combine all three graph components into a single `networkx.DiGraph` object and create a unique node identifier for each node in the combined graph.
3. Split the combined graph into the three *output* graph components again (or more if the specific graph architecture requires it).
4. Store each of the *output* graph components in the desired format, for example:

    - **pytorch-geometric**: edges indexes and features are stored in separate `torch.Tensor` objects serialised to disk that can then be loaded into `torch_geometric.data.Data` objects
    
### Diagram of the graph generation process:

```mermaid
graph TB

xy_grid["(x,y) grid coordinates [numpy.ndarray]"]

subgraph weather_model_graphs["weather-model-graphs"]

    subgraph graph_components["Graph components"]
        G_g2m["G_g2m[networkx.DiGraph]"]
        G_m2m["G_m2m[networkx.DiGraph]"]
        G_m2g["G_m2g[networkx.DiGraph]"]
    end

    G_g2m --merged into --> G_full
    G_m2m --merged into --> G_full
    G_m2g --merged into --> G_full

    G_full["complete graph\nG_full[networkx.Digraph]"]

    G_full --nodes labels given unique identifiers--> G_full_int["G_int[networkx.Digraph]"]

    G_full_int --split and converted into --> pyg_g2m["pyg_g2m[pyg.data.Data]"]
    G_full_int --split and converted into --> pyg_m2m["pyg_m2m[pyg.data.Data]"]
    G_full_int --split and converted into --> pyg_m2g["pyg_m2g[pyg.data.Data]"]
end

xy_grid --used to create grid nodes in --> G_g2m
xy_grid --used to create grid nodes in --> G_m2g
xy_grid --used to create grid nodes in --> G_m2m

subgraph stored_on_disk["stored on disk"]
    pyg_g2m_file["g2m_edge_index.pt\ng2m_features.pt"]
    pyg_m2m_file["m2m_edge_index.pt\nm2m_features.pt"]
    pyg_m2g_file["m2g_edge_index.pt\nm2g_features.pt"]
end

pyg_g2m --stored in--> pyg_g2m_file
pyg_m2m --stored in--> pyg_m2m_file
pyg_m2g --stored in--> pyg_m2g_file

subgraph pyg_loaded["Loaded into model (e.g. torch.nn.Module)"]
    pyg_g2m_loaded["pyg_g2m[pyg.data.Data]"]
    pyg_m2m_loaded["pyg_m2m[pyg.data.Data]"]
    pyg_m2g_loaded["pyg_m2g[pyg.data.Data]"]
end

pyg_g2m_file --loaded into--> pyg_g2m_loaded
pyg_m2m_file --loaded into--> pyg_m2m_loaded
pyg_m2g_file --loaded into--> pyg_m2g_loaded
```

### Node and edge attributes

There are a number of node and edge attributes with special meanings in `weather-model-graphs` which enable the splitting and visualisation of the graph components.

#### Node attributes

- `pos`: the `(x,y)` coordinates of the node in the grid
- `type`: the type of node, either `grid` or `mesh`


#### Edge attributes

- `component`: the component of the graph the edge belongs to, either `g2m`, `m2m` or `m2g`
- `level`: for multi-range mesh graphs this denotes the refinement level of mesh connection. For hierarchical graphs the different ranges of connections are split into different levels and so here `level` also denotes the level in the hierarchy that the edge belongs to.
- `len`: the length of the edge in the (x,y) coordinate space of the grid nodes, i.e. the distance between the two nodes in the grid
- `direction`: for hierarchical graphs this denotes the direction of the edge, either `up`, `down` and `same`


### Splitting graphs

The splitting of the graph is done with by utilising the edge attributes, and thus it is easy to split the complete graph by either which component the edge belongs to, or by the level of the edge in the graph. This is done using the `weather_model_graphs.split_graph_by_edge_attribute(...)` function.