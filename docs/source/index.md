import osmnx as ox
import networkx as nx
from shapely.geometry import LineString
import random

# Location of the testing center
TESTING_CENTER = (51.5074, -0.1278)  # Example: London coords

# Download the street network (you can use a custom polygon)
G = ox.graph_from_point(TESTING_CENTER, dist=2000, 
                        network_type='drive', simplify=True)

# Remove disallowed roads
allowed_highway_types = [
    'residential', 'tertiary', 'secondary', 'primary', 
    'unclassified', 'living_street', 'service'
]
G = ox.utils_graph.graph_from_gdfs(
    *ox.graph_to_gdfs(G, nodes=True, edges=True)
)

# Filter edges (roads)
edges = ox.graph_to_gdfs(G, nodes=False)
edges = edges[edges['highway'].apply(lambda x: any(hw in allowed_highway_types for hw in (x if isinstance(x, list) else [x])))]

# Rebuild filtered graph
G = ox.graph_from_gdfs(*ox.graph_to_gdfs(G, edges=edges))

# Choose start node nearest the testing center
start_node = ox.nearest_nodes(G, TESTING_CENTER[1], TESTING_CENTER[0])

def is_valid_route(route, min_km=6, max_km=8):
    # Ensure length within bounds and diversity of roads
    total_length = sum(ox.utils_graph.get_route_edge_attributes(G, route, 'length'))
    if total_length < min_km * 1000 or total_length > max_km * 1000:
        return False
    # Analyze road types
    road_types = set()
    for u, v in zip(route[:-1], route[1:]):
        data = G.get_edge_data(u, v)
        if data:
            highway = data[0].get('highway')
            if isinstance(highway, list):
                road_types.update(highway)
            else:
                road_types.add(highway)
    # Roundabouts detection, simple version
    roundabouts = [data for u, v, data in G.edges(data=True) if 'junction' in data and data['junction'] == 'roundabout']
    has_roundabout = any(edge for edge in roundabouts if edge['u'] in route and edge['v'] in route)
    return (
        has_roundabout and
        'residential' in road_types and
        'primary' in road_types
    )

# Generate candidate looped routes
routes = []
for i in range(50):
    try:
        loop = nx.shortest_path(G, start_node, start_node, weight='length', method='dijkstra')
        if is_valid_route(loop):
            routes.append(loop)
    except:
        continue

# Plot one
ox.plot_graph_route(G, routes[0], route_linewidth=4, node_size=0)
