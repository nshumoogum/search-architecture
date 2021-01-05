# search-architecture

Source of truth for backend processing of search data for ONS sitewide search.

### Sequence Diagrams

The source script for the each diagram is maintained in sequence folder.

To update a diagram:
- Copy the source script into [sequencediagram.org](http://sequencediagram.org)
- Update a diagram as required
- Export as an image on [sequencediagram.org](http://sequencediagram.org)
- Copy the updated source script back into `sequence/<name>.txt` and commit.

#### Add/update search documents when a collection is published

![Publish Search Data](sequence/PublishSearchData.png)

#### Reindex

![Reindex Search](sequence/ReindexSearch.png)