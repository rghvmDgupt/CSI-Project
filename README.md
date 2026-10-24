# Road Network and Initial Conditions

**Project:** Iterative Road Quality Estimation\
**Module owner:** Raghavam\
**Module:** `road_network.py`

## 1. Purpose

This module creates a configurable simulated road network for the team's
iterative road-quality estimation project. Intersections are represented
as nodes, and roads are represented as segments connecting pairs of
nodes.

Each road segment has a unique ID, a road class, a length, a speed
limit, an initial roughness estimate, and a hidden simulated
ground-truth roughness value.

Roughness values are normalized simulation scores in the range `[0, 1]`.
They are **not measured IRI values**.

## 2. Scope

This module is responsible for:

-   Generating a connected road graph.
-   Assigning unique IDs to road segments.
-   Classifying roads as highways, main roads, or small roads.
-   Assigning initial and hidden true roughness values.
-   Providing graph access and structural validation.
-   Supporting reproducible experiments through a random seed.

Vehicle-observation generation, observation aggregation, iterative
estimation, baseline comparison, robustness evaluation, parameter
tuning, and visualization belong to other project modules.

## 3. Dependencies

The module uses Python standard-library packages only:

-   `dataclasses`
-   `random`
-   `typing`
-   `math` (remove this import if it is unused in the final source)

No external packages are required for network generation.

## 4. Data structures

### `RoadSegment`

Each `RoadSegment` object contains:

  -----------------------------------------------------------------------
  Field                               Description
  ----------------------------------- -----------------------------------
  `segment_id`                        Unique road-segment ID, such as
                                      `R00001`

  `start_node`                        ID of the first endpoint

  `end_node`                          ID of the second endpoint

  `road_class`                        `highway`, `main_road`, or
                                      `small_road`

  `length_km`                         Simulated segment length in
                                      kilometres

  `initial_roughness`                 Starting roughness estimate in
                                      `[0, 1]`

  `true_roughness`                    Hidden simulated ground-truth
                                      roughness in `[0, 1]`

  `speed_limit_kmph`                  Simulated speed limit in km/h
  -----------------------------------------------------------------------

### `RoadNetwork`

The network stores:

-   `nodes`: a set of intersection IDs.
-   `segments`: a dictionary mapping each segment ID to its
    `RoadSegment`.
-   `adjacency`: a dictionary mapping each node to
    `(neighbor_node, segment_id)` entries.

The current generator models roads as bidirectional and does not create
self-loops or duplicate edges.

## 5. Road-class assumptions

  -----------------------------------------------------------------------
  Road class         True roughness   Initial estimate        Speed limit
                              range              prior 
  -------------- ------------------ ------------------ ------------------
  Highway                0.05--0.35               0.15           100 km/h

  Main road              0.20--0.65               0.40            50 km/h

  Small road             0.35--0.90               0.60            25 km/h
  -----------------------------------------------------------------------

These are simulation assumptions, not universal claims about real roads.
The initial estimate is perturbed by `initial_noise` and clipped to
`[0, 1]`. Ground truth is sampled from the selected class's roughness
range.

## 6. Main function and parameters

``` python
generate_road_network(
    num_nodes=50,
    num_segments=80,
    seed=42,
    initial_noise=0.15,
    class_weights=None,
)
```

  -----------------------------------------------------------------------
  Parameter                           Meaning
  ----------------------------------- -----------------------------------
  `num_nodes`                         Number of intersections; must be at
                                      least 1

  `num_segments`                      Number of road segments; must be at
                                      least `num_nodes - 1` for
                                      connectivity

  `seed`                              Seed used for random generation

  `initial_noise`                     Maximum absolute deviation from the
                                      class-based initial estimate; must
                                      be non-negative

  `class_weights`                     Optional relative weights for the
                                      three road classes
  -----------------------------------------------------------------------

If `class_weights` is omitted, the defaults are highway `0.25`, main
road `0.45`, and small road `0.30`. These are selection weights, not
guaranteed final proportions.

The generator raises `ValueError` for invalid network sizes, invalid
class weights, or negative initial noise.

## 7. Useful methods

  ---------------------------------------------------------------------------
  Method                                  Purpose
  --------------------------------------- -----------------------------------
  `add_node(node_id)`                     Adds an intersection

  `add_segment(segment)`                  Adds a road after checking basic
                                          constraints

  `get_segment(segment_id)`               Returns the requested `RoadSegment`

  `get_neighbors(node_id)`                Returns neighboring nodes and
                                          connecting segment IDs

  `get_segments_by_class(road_class)`     Returns segments belonging to a
                                          road class

  `is_connected()`                        Checks whether all nodes are
                                          reachable

  `validate()`                            Checks core graph and reference
                                          consistency

  `summary()`                             Returns node count, segment count,
                                          connectivity, validity, and class
                                          counts

  `get_observation_context(segment_id)`   Returns road metadata without
                                          exposing `true_roughness`

  `get_simulation_truth(segment_id)`      Returns hidden truth for
                                          simulation/evaluation use only
  ---------------------------------------------------------------------------

`get_simulation_truth()` is for controlled simulation and evaluation. It
must not be included as an input feature for the iterative estimator.

## 8. Example usage

Save `road_network.py` in your project directory and run:

``` python
from road_network import generate_road_network

network = generate_road_network(
    num_nodes=50,
    num_segments=80,
    seed=42,
)

print(network.summary())

segment = network.get_segment("R00001")
print(segment)

context = network.get_observation_context("R00001")
print(context)  # Does not include true_roughness
```

The observation context includes the segment ID, class, length, speed
limit, and initial roughness estimate. It deliberately excludes hidden
ground truth.

## 9. Unit tests

The test notebook is `test_road_network.ipynb`. It currently contains 10
unit tests covering:

-   Requested network size.
-   Connectivity and structural validity.
-   Unique segment IDs.
-   Presence of road classes.
-   Roughness bounds.
-   Bidirectional adjacency.
-   Same-seed reproducibility within the current environment.
-   Rejection of impossible network sizes.
-   Rejection of duplicate segment IDs.

The tests were observed passing in the Jupyter session:

``` text
Ran 10 tests
OK
```

Run them in Jupyter by executing the test notebook's test cell. For a
conventional Python module and test file, you can instead run:

``` bash
python -m unittest -v test_road_network.py
```

Note: if your tests currently exist only in `test_road_network.ipynb`,
this command will work only after the tests are saved in a Python file
named `test_road_network.py`.

## 10. Experiment record

The following results were observed in the experiment notebook:

  --------------------------------------------------------------------------------
     Nodes   Segments     Seed  Connected    Valid      Highways     Main    Small
                                                                    roads    roads
  -------- ---------- -------- ----------- ---------- ---------- -------- --------
        20         30       42    True        True             5       17        8

        50         80       42    True        True            15       38       27

       100        160       42    True        True            38       71       51

        50         80      100    True        True            21       38       21
  --------------------------------------------------------------------------------

All four tested networks were reported as connected and valid.
Road-class counts vary with the random seed because classes are sampled
probabilistically.

The experiment checks network generation, connectivity, validity, and
class distribution. It does not measure road-quality estimation
accuracy; that evaluation belongs to the relevant project modules.

## 11. Important limitations and assumptions

-   The graph is an abstract simulated graph; it does not use real
    geographic coordinates.
-   Roads are bidirectional.
-   Segment lengths and speed limits are synthetic.
-   Road-quality values are normalized simulation scores, not measured
    IRI.
-   The class-based ranges and initial priors are design assumptions for
    the simulation.
-   Hidden ground truth is included so other modules can generate
    controlled synthetic observations and evaluate estimators. The
    estimator must not receive it as an input.
-   The generator ensures graph connectivity by first creating a chain
    and then adding extra edges.

## 12. Deliverable checklist

-   [x] Simulated road network with unique segment IDs.
-   [x] Highway, main-road, and small-road classifications.
-   [x] Initial roughness and hidden ground-truth values.
-   [x] Graph data structure and connectivity check.
-   [x] Ten unit tests passed in the recorded Jupyter run.
-   [x] Experiments with 20, 50, and 100 nodes, plus a different seed.
-   [x] Documentation of inputs, outputs, assumptions, and dependencies.

**Owner:** Raghavam\
**Primary module:** `road_network.py`\
**Test notebook:** `test_road_network.ipynb`
