---
layout: post
title:  "Filtering and updating a set of rows in pandas"
date:   2024-08-01 15:19:51 -0700
categories: pandas
---
While working with pandas I had a use case where I had to filter a set of rows and update the values on all those rows at once. Doing apply was slow as I had to do the operation on a set of rows (think vectorization).

There was no good article on how to do that. So here goes my approach
1. Filter the rows that you need to process. In my case, the column I needed to filter was named `embedding`

{% highlight python %}
is_row_new = df["embedding"].isnull()
new_rows = df[is_row_new]
{% endhighlight %}

2. I had to get the embedding for these rows as a batch. I stored these list of embeddings as `new_embeddings` 
{% highlight python %}
new_embeddings: list = create_new_embeddings(new_rows)
{% endhighlight %}

3. Create a pandas `Series` and apply it back to the data frame
{% highlight python %}
new_embedding_updates = pd.Series(new_embeddings, index=df.index[is_row_new])
df["embedding"].update(new_embedding_updates)
{% endhighlight %}

That's it - the trick with dealing with a `list` is essentially to keep an index of all the rows that you've to update. Once you have the new values, just create a new `Series` and update back the dataframe.

## Full Snippet

{% highlight python %}
is_row_new = df["embedding"].isnull()
new_rows = df[is_row_new]
new_embeddings: list = create_new_embeddings(new_rows)
new_embedding_updates = pd.Series(new_embeddings, index=df.index[is_row_new])
df["embedding"].update(new_embedding_updates)
{% endhighlight %}
