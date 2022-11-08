# Research_findings
## Graph Preprocess
1. Get AKL bounding box
2. Get POIs from bounding box and project to UTM Zone(default)
    - take osm node id from projected df as new Id
    - output akl_projected_poi_node_df_with_ways.csv
3. Get roads from bounding box and project to UTM Zone(default)
    - for Polygon, MultiPolygon and LineString type node, take x,y of the centroid as coordinates
4. Find edge for POIs
    - Pass in Road Graph and raw POI DF
    - Find nearest_edges for all the POIs by calling osmnx api. 
    - Loop through POI DF
        - calculate distance from POI to nearest road's start point
        - calculate distance from POI to nearest road's end point
        - New from nearest road (source,target,distance_to_edge) + Existing POI information (amenity, name, geometry, shop, clothes, x, y)
    - Save to akl_poi_mapping.csv

5. Construct Street Nodes
    - Pass in Road Graph and processed POI DF
    - For every edge of roads
        - If type is reversed or edge does not have a name
            - SKIP
        - From POI DF, find all the pois with same (source, target) of the current edge (If not found, do reverse (target, source))
        - Loop through every poi and their features
            - Aggregate distance for every matched POI to current edge
            - Count up different types of POIs and rename some e.g (clinic,hospital) -> healthcare
        - Divide Aggregated distance by number of pois to get average distance to POI for the road edge.
    - Fill distance for road edges without POIs to the length of the road
    - Rename index to street_id
    - Output to akl_street_nodes.csv

6. Construct Street Edges
    - Pass in processed Street Nodes
    - For every street node
        - Find other street with the same source and different target
        - Find other street with the same target and different source
        - Edge Distance  = street distance + neighbour street distance
    - Output akl_street_edges.csv

## Property Preprocess
1. Load Cleaned Property Sale Data and AKL Street Nodes csv
2. Get AKL bounding box
3. Map every property's coordianates to CRS
4. Get roads from bounding box and project to UTM Zone(default)
    - for Polygon, MultiPolygon and LineString type node, take x,y of the centroid as coordinates
5. For every property, find it's nearest edge and record the (street_id,source,target) of the edge
6. Merge with Cleaned Property Sale Data and generate property_data_with_street.csv


## GraphSAGE Data Load
1. load akl_street_nodes.csv and drop unused features (coordinates for example)
2. load akl_street_edges.csv 

### GraphSAGE Sampling
#### Without POI
Maintain a list of postive node sequence
1. For every node in a batch
    - Find the neighbours from edge DF using the node id 
    - Find all the neighbours weights(distances) and convert into range of (0,1) normalized
    - **IF NO Neighbours**
        - add self to the sequence eg.[1,1] means node 1 -> node 1
        - continue
    - **ELSE**
        - Random select neighbour based on the previous normalized probability
        - Get the next edge information from Edge DF
        - Check the (source,target) for new edge
            - Take the one does not match the current node
            - Append to the sequence e.g [1,2] means node 1 -> node 2

#### With POI
Maintain four lists
- positive node sequence -> List of positive nodes 
- negative node sequence -> List of negative nodes
- POI NODES SET -> List of Nodes containing POIs 
- NO POI NODES SET -> List of Nodes dont have POIS

When current node in the batch does not have any neighbours have POIs, Its negative sample will be nodes with POI.

When current node in the batch have neighbours with POIs, Its negative sample will be nodes without POI.

(First few run will always random pick the negative sample)

1. For every node in a batch
    - Find the neighbours from edge DF using the node id 
    - For every neighbours, check the (source,target)
            - Take the one does not match the current node and call it neighbour_id_list
    - For every neighbour id in the neighbour_id_list
        - init weight to 0
        - Find street nodes matched with neighbour id
        - Sum the feature of matched nodes (Sum up number of POIs on the neighbour node)
        - Append weight into a weight_list
    - Convert weight_list into range of (0,1) normalized
    - **IF NO NEIGHBOURS HAVE POIS)**
        - Add current node in to No POI nodes SET.
        - **IF NO NEIGHBOURS (Isolated nodes)** 
            - add self to the sequence eg.[1,1] means node 1 -> node 1
            - **IF POI NODES SET IS EMPTY**
                - Random pick one from street nodes and add to negative node sequence
            - **ELSE**
                - Pick one from POI NODES SET and add it to negative node sequence.
            - Continue
        - **ELSE NUMBER OF NEIGHBOURS NOT EMPTY**
            - Random pick one from neighbour
    - **ELSE NEIGHBOURS GOT WEIGHTS**
        - Add current node to POI nodes SET
        - Random pick using distribution of POI weights
    - Get the next edge information from Edge DF
    - Check the (source,target) for new edge
        - Take the one does not match the current node
    - Append to the positive sequence e.g [1,2] means node 1 -> node 2
    - **IF NO POI NODES SET IS EMPTY**
        - Random pick one from street nodes and add to negative node sequence
    - **ELSE**
        - Pick one from NO POI NODES SET and add it to negative node sequence

本来打算在每一层走的时候，Look ahead N层然后生成edge的选取概率,没有做下去有两个原因
1. 计算量大
2. 好像和GRAPHSAGE的概念重合了 等于做了两次GRAPHSAGE?

比如会预处理N个文件，节点1层，节点2层，节点N层
- 节点1层就和现在一样只看当前节点的POI
- 节点2层的特征变为自己加上邻居1层的POI总量
- .....

带POI的采样的时候没有考虑进边的距离，或许在算概率分布的时候可以把边的距离想办法加个权和POI一起合为一个概率?


### GraphSAGE
1. N SAGEConv with dropout 
2. Use Adam to optimize the process
