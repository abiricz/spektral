## Representing graphs

Spektral uses a matrix-based representation for manipulating graphs and feeding them to neural networks. This approach is one of the most commonly used in the literature on graph neural networks, and it's perfect to perform parallel computations on GPU.

A graph is generally represented by three matrices:

- a binary adjacency matrix, \(A \in \{0, 1\}^{N \times N}\), where \(A_{ij} = 1\) if there is a connection between nodes \(i\) and \(j\), and \(A_{ij} = 0\) otherwise;
- a matrix encoding node attributes, \(X \in \mathbb{R}^{N \times F}\), where each row represents the \(F\)-dimensional attribute vector of a node;
- a matrix encoding edge attributes, \(E \in \mathbb{R}^{N \times N \times S}\), where each entry represents the \(S\)-dimensional attribute vector of an edge;

Some formulations (like the [graph networks](https://arxiv.org/abs/1806.01261) proposed by Battaglia et al.) also include a feature vector describing the global state of the graph, but this is not supported by Spektral for now. 

In code, the three matrices described above are Numpy arrays `A`, `X`, and `E`, where `A.shape == (N, N)`, `X.shape == (N, F)`, and `E.shape == (N, N, S)`.
Sparse adjacency matrices are also supported by Spektral.


## Modes

In Spektral, some functionalities are implemented to work on a single graph, while others consider sets (i.e., batches) of graphs. 

To understand the need for different settings, consider the difference between classifying the nodes of a citation network, and classifying the chemical properties of molecules.
In the citation network, we are interested in the single nodes and the connections between them. Node and edge attributes are specific to each individual network, and there is no point in training a model across networks. 
On the other hand, when working with molecules in a dataset, we are in a much more familiar setting. Each molecule is a sample of our dataset, and the atoms and bonds that make up the molecules are repeated across the data (like pixels in images). In this case, we are interested in finding patterns that describe the properties of the molecules in general. 

The two settings require us to do things that are conceptually similar, but that need some minor adjustments in how the data is processed by our graph neural networks. This is why Spektral makes these differences explicit.


In practice, we actually distinguish between three main _modes_ of operation: 

- **single**, where we have a single graph, with its topology and attributes;
- **batch**, where we have a collection of graphs, each with its own topology and attributes;
- **mixed**, where we have a graph with fixed topology, but a collection of different attributes (usually called _graph signals_ in the literature); this can be seen as a particular case of the batch mode (where all adjacency matrices are the same) but it is handled separately in Spektral to improve memory efficiency.

|Mode  |Adjacency    |Node attr.   |Edge attr.      |
|:----:|:-----------:|:-----------:|:--------------:|
|Single|(N, N)       |(N, F)       |(N, N, S)       |
|Batch |(batch, N, N)|(batch, N, F)|(batch, N, N, S)|
|Mixed |(N, N)       |(batch, N, F)|(batch, N, N, S)|


In practice, in "single" mode the data has no `batch` dimension, and describes a single graph:

```py
In [1]: from spektral.datasets import citation
Using TensorFlow backend.

In [2]: adj, node_features, _, _, _, _, _, _ = citation.load_data('cora')
Loading cora dataset

In [3]: adj.shape
Out[3]: (2708, 2708)

In [4]: node_features.shape
Out[4]: (2708, 1433)
```

This means that when training GNNs in single mode, we cannot batch and shuffle the data along the first axis, and the whole graph must be fed to the model at each step (see [the node classification example](https://github.com/danielegrattarola/spektral/blob/master/examples/node_classification_gcn.py)).

In "batch" mode, the matrices will have a `batch` dimension first: 

```py
In [1]: from spektral.datasets import qm9
Using TensorFlow backend.

In [2]: adj, nf, ef, _ = qm9.load_data()
Loading QM9 dataset.
Reading SDF
100%|█████████████████████| 133885/133885 [00:29<00:00, 4579.22it/s]

In [3]: adj.shape
Out[3]: (133885, 9, 9)

In [4]: nf.shape
Out[4]: (133885, 9, 6)

In [5]: ef.shape
Out[5]: (133885, 9, 9, 1)
```

This should not surprise you if you are familiar with classical machine learning tasks (try and load any benchmark image dataset to see a similar representation of the data).

Finally, in "mixed" mode we consider a single adjacency matrix with no `batch` dimension, and collections of node and edge attributes with the `batch` dimension:

```py
In [1]: from spektral.datasets import mnist
Using TensorFlow backend.

In [2]: nf, _, _, _, _, _, adj = mnist.load_data()

In [3]: adj.shape
Out[3]: (784, 784)

In [4]: nf.shape
Out[4]: (50000, 784)
```

### Graph batch mode

![](https://danielegrattarola.github.io/spektral/img/graph_batch.svg)

|WARNING|
|:------|
| Note that support for edge attributes is not yet implemented in Spektral |

When dealing with graphs with a variable number of nodes, representing a group of graphs in batch mode requires padding `A`, `X`, and `E` to a fixed dimension.  
In order to avoid this issue, a common approach is to represent a batch of graphs by taking their disjoint union, and then working with this "supergraph" in single mode.

In order to compute the disjoint union of a batch of graphs: 

1. We take a block diagonal matrix constructed from the adjacency matrices in the batch;
2. We stack node features together along the node dimension;
3. We take a block diagonal matrix constructed from the edge features matrices in the batch (if S > 1, we compute a block diagonal matrix for each element of the features, and then stack those along the last dimension).

Additionally, we also keep track of which nodes belong to which graphs by having a vector of integer indices mapping each node to a number between 0 and the batch size.  
Utilities for dealing with this alternative representation of graph batches are provided in `spektral.utils.data`:

```py
In [1]: from spektral.utils.data import Batch                                                               
Using TensorFlow backend.

In [2]: A_list = [np.ones((2, 2))] * 3                                                                

In [3]: X_list = [np.random.normal(size=(2, 4))] * 3                                                  

In [4]: b = Batch(A_list, X_list)                                                                     

In [5]: b.A.todense()                                                                                 
Out[5]: 
matrix([[1., 1., 0., 0., 0., 0.],
        [1., 1., 0., 0., 0., 0.],
        [0., 0., 1., 1., 0., 0.],
        [0., 0., 1., 1., 0., 0.],
        [0., 0., 0., 0., 1., 1.],
        [0., 0., 0., 0., 1., 1.]])

In [6]: b.X                                                                                           
Out[6]: 
array([[-0.85196709, -1.66795384, -1.14046868,  0.4735151 ],
       [ 0.14221143, -0.76473164, -1.05635638,  1.45961459],
       [-0.85196709, -1.66795384, -1.14046868,  0.4735151 ],
       [ 0.14221143, -0.76473164, -1.05635638,  1.45961459],
       [-0.85196709, -1.66795384, -1.14046868,  0.4735151 ],
       [ 0.14221143, -0.76473164, -1.05635638,  1.45961459]])

In [7]: b.I                                                                                           
Out[7]: array([0, 0, 1, 1, 2, 2])

In [8]: b.get('AXI')                                                                                  
Out[8]: 
(<6x6 sparse matrix of type '<class 'numpy.float64'>'
  with 12 stored elements in COOrdinate format>,
 array([[-0.85196709, -1.66795384, -1.14046868,  0.4735151 ],
        [ 0.14221143, -0.76473164, -1.05635638,  1.45961459],
        [-0.85196709, -1.66795384, -1.14046868,  0.4735151 ],
        [ 0.14221143, -0.76473164, -1.05635638,  1.45961459],
        [-0.85196709, -1.66795384, -1.14046868,  0.4735151 ],
        [ 0.14221143, -0.76473164, -1.05635638,  1.45961459]]),
 array([0, 0, 1, 1, 2, 2]))
```

Convolutional layers that work in single mode will work for this type of data representation without any modification.  
Pooling layers, on the other hand, require the batch indices vector in order to know which nodes to pool together. 
Global pooling layers do not need the indices vector, and they simply operate on X. 
Standard pooling layers, on the other hand, will also return a reduced version of the vector.

---

## Conversion methods

To provide better compatibility with other libraries, Spektral has methods to convert graphs between the matrix representation (`'numpy'`) and other formats. 

The `'networkx'` format represents graphs using the Networkx library, which can then be used to [convert the graphs](https://networkx.github.io/documentation/networkx-1.10/reference/convert.html) to other formats like `.dot` and edge lists. 
Conversion methods between `'numpy'` and `'networkx'` are provided in `spektral.utils.conversion`.

---

## Molecules

When working with molecules, some specific formats can be used to represent the graphs. 

The `'sdf'` format is an internal representation format used to store an SDF file to a dictionary. A molecule in `'sdf'` format will look something like this: 

```py
{'atoms': [{'atomic_num': 7,
           'charge': 0,
           'coords': array([-0.0299,  1.2183,  0.2994]),
           'index': 0,
           'info': array([0, 0, 0, 0, 0, 0, 0, 0, 0, 0]),
          'iso': 0},
          ...,
          {'atomic_num': 1,
           'charge': 0,
           'coords': array([ 0.6896, -2.3002, -0.1042]),
           'index': 14,
           'info': array([0, 0, 0, 0, 0, 0, 0, 0, 0, 0]),
           'iso': 0}],
'bonds': [{'end_atom': 13,
           'info': array([0, 0, 0]),
           'start_atom': 4,
           'stereo': 0,
           'type': 1},
          ...,
          {'end_atom': 8,
           'info': array([0, 0, 0]),
           'start_atom': 7,
           'stereo': 0,
           'type': 3}],
'comment': '',
'data': [''],
'details': '-OEChem-03231823253D',
'n_atoms': 15,
'n_bonds': 15,
'name': 'gdb_54964',
'properties': []}
```

The `'rdkit'` format uses the [RDKit](http://www.rdkit.org/docs/index.html) library to represent molecules, and offers several methods to manipulate molecules with a chemistry-oriented approach.

The `'smiles'` format represents molecules as strings, and can be used as a space-efficient way to store molecules or perform quick checks on a dataset (e.g., counting the unique number of molecules in a dataset is quicker if all molecules are converted to SMILES first).

The `spektral.chem` modules offers conversion methods between all of these formats, although some conversions may need more than one step to do (e.g., `'sdf'` to `'networkx'` to `'numpy'` to `'smiles'`). Support for direct conversion between all formats will eventually be added.