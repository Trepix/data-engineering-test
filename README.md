# data-engineering-test

## Problem
There are different aspects to consider: data ingestion, validation, normalization, consolidation, and exploitation. The solution for each of those is conditioned by the assumptions I made. In another context, instead of assuming, I would explore the data, extrapolate/project data from other scenarios, or ask the Product Owner.

## Assumptions / Constraints
* Data from partner platforms have different life cycles. Some are updated every hour, others daily. Each partner has its own data freshness.
* Updates are heterogeneous. Some are partial, like this movie now has this rating; others are snapshots that provide all the info and any field may change.
* Speed ​​is not a requirement. Having a window of hours between receiving the info and serving it is not a problem. 
* Load is not a problem. Some executions may need to handle more data but for example, we don't receive thousands of new records at 00:00 which would require specific orchestration requirements.
* Exists some kind of mechanism that if the main source (to determine a title or genre) is not available, the C-Rating™ algorithm can work well (chain of sources or work with older data)
* C-Rating™ algorithm needs all the information from partners to generate the output, not partial or aggregated info.
* C-Rating™ can normalize the distribution between partner ratings. For example when a platform has a higher scoring tendency than others overall.
* The tech stack is not the most important part of this test, so I will mention only some tools and not describe the trade-offs.

## Phases
### Ingestion
In this phase, we only add external data to our platform without any transformation. The first reason for doing this is if an error is raised, we can track down the issue and determine whether it originates from our system or the imported data. Second, if the problem is not in the data, we can rebuild the latest state from the same data after fixing the code.

Pull APIs store the records in files with JSONL format in S3 for each execution. Push APIs can use Kafka as a buffer to avoid the “small files problem” and end up writing partitioned by hour to apply purge or compact processes.
FTP and other batch processes will move the file into our ingestion area.

_From this phase onward, we orchestrate the processes using Airflow. The required freshness of the data, its volume, potential rollbacks, or reprocess needs, define most of the approaches to the problem.
For our assumptions, and going for simplicity all data from ingestion to consolidation is sequentially processed in batches to generate an hourly (maybe bi-hourly) snapshot._


### Validation
Here’s another trade-off for simplicity: doing it in a separate step from ingestion simplifies the workflow but delays the error detection until this phase runs. I advocate for shifting left quality problems but in this case, I prefer to keep it simple. Unifying both logic (ingest + validate) could increase complexity in a non-linear way.
After each ingestion, we handle invalid data format errors. We only check if data can be read, has the expected type, and it’s in the expected ranges. If files or data sources are not readable we discard the whole source ingestion, but if only a few records are invalid only these are discarded.

Data is stored in a row format under another S3 prefix (or folder).


### Normalization
As stated in the problem description, the data is heterogeneous (different genres across platforms, and the same movie can have multiple, non-matching genres). In this phase, the data is normalized so the C-Rating™ algorithm does not have to deal with this complexity. 

Titles and tags are assigned based on the source of truth chosen for each movie.  This could be done using fuzzy logic and discarding records that do not meet a threshold of resemblance. Although this approach complicates the solution, it ensures that the data meets a minimum quality and allows us to detect possible new cases of heterogeneity. For example, if our reliable partner for a movie tags a movie as “drama”, we can admit “war drama” as a similar tag. But, perhaps, a movie is tagged as an “emotional narrative”. This tag, initially, could not be related to drama, but then, after looking over it, maybe we will admit it as a valid synonym. Then, with our approach, we have been able to detect a new case of heterogeneity. 
On the opposite side, if another partner tags the same movie as Comedy this is not a semantic difference, maybe we have to check what's going on with our source of truth or our mistagging partner.


In this phase, the units of the ratings are also normalized, for example, by using the system with the highest precision (stars systems 1-5 vs percentage system 0-100).  We also extrapolate or calculate the movie’s rating for each category (or similar) used in the C-Rating™ algorithm to generate the output rating (performance, screenplay, and soundtrack). For example, if a platform splits the screenplay score between the original and the adapted, and the C-Rating™ algorithm doesn't matter about this distinction and only needs a number for the screenplay category we average them. Finally, if a partner doesn't provide any of these categories and the C-Rating™ algorithm doesn't know how to deal with that we could use the total score on each category.

However, as we said in the assumptions epigraph, we assume the C-Rating™ will handle the biases between platforms (p.eg: a platform that persistently overrates the movies). So we do not try to manage that in this phase

In my opinion, this is a really interesting part but time and length limitations lead me not to go into it further.


### Consolidation
C-Rating™ algorithm uses the output of this phase. We separate this phase from the previous one to, once again, track any potential problems in normalization. Especially in the step of linking similar tags and checking if we are missing any potential relations or considering very different genres as equivalent.
This data represents a snapshot of the current state of all partner platforms, versioned by execution, allowing us to perform a rollback and reprocess changes from a specific point in case of an error.
Data is stored under another S3 prefix (or folder) but its format would depend on the C-Rating™ algorithm needs. If more processes must rely on this data we might need to change and rethink this approach.

### Exploitation 
We have listeners waiting for the output of the C-Rating™ algorithm. When the process finishes, the data is stored in a query engine such as Solr or MySQL, depending on the query requirements. This strategy allows us to adapt to UX requirements without modifying any code in the C-Rating™ algorithm.

![img](./scheme.png)

