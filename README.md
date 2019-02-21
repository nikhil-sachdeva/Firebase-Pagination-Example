# Firebase-Pagination-Example
My implementation of pagination in Firebase <br>
### Original Article [Link](https://mostly-dumb.github.io/firebase-pagination/)
---
project: true
layout: post
title: "Firebase Database Pagination"
date: 2019-02-18
excerpt: "Tutorial for implementing pagination in Firebase."
---

## What is Pagination?
While loading data from an API on internet connection, it is important to load it in batches as and when required. If Facebook/Instagram were to load all the data altogether, the apps would never start. Hence more data should be loaded only when more data is requested i.e. when the screen is scrolled further. This is the efficient practice in any API loading scenario. This also uses internet and cache memory efficiently.

## What happens in Firebase?
Unfortunately, in most of the tutorials and the Firebase Documentation found online for receiving data, all the data is loaded at once.
This is the Firebase Documentation for retrieving lists of data.
~~~ java
myTopPostsQuery.addValueEventListener(new ValueEventListener() {
    @Override
    public void onDataChange(DataSnapshot dataSnapshot) {
        for (DataSnapshot postSnapshot: dataSnapshot.getChildren()) {
            // TODO: handle the post
        }
    }

    @Override
    public void onCancelled(DatabaseError databaseError) {
        // Getting Post failed, log a message
        Log.w(TAG, "loadPost:onCancelled", databaseError.toException());
        // ...
    }
});
~~~

Now this doesn't work for a long list of data in reasonable time for a smooth experience.

## Solution

Considering an example of list of posts to be viewed in RecyclerView, following are the steps to implement pagination in Firebase Realtime Database.
#### The solution is simple and explained below :
1. Define mPosts, an integer constant, the number of posts to be viewed at once on the screen.
2. Retrieve the first batch of posts and pass it to the RecyclerView Adapter.
3. Override the onScrolled method of the RecyclerView to load more data upon scrolling the RecyclerView.
4. Use LinearLayoutManager to find the last fully visible item of the list.
5. Load more data just before the last item is to be scrolled.
6. Add the loaded data in the ArrayList<T> of object. And subsequently notify the adapter that the data set has changed.
7. To maintain unique entries, check if the ArrayList has unique objects of class Post. For this, "equals" method must be overridden in the Posts class.


Now, let's code this solution.

## Code
We'll implement this solution for a list of posts to be viewed in RecyclerView.
Following is the Post POJO model class in which the equals method has been overridden:
~~~ java
public class Post {
    String topic, details, imageURL;
    Date date;


    public String getTopic() {
        return topic;
    }

    public void setTopic(String topic) {
        this.topic = topic;
    }

    public String getDetails() {
        return details;
    }

    public void setDetails(String details) {
        this.details = details;
    }

    public String getImageURL() {
        return imageURL;
    }

    public void setImageURL(String imageURL) {
        this.imageURL = imageURL;
    }

    public String getDate() {
        return date;
    }

    public void setDate(Date date) {
        this.date = date;
    }

    @Override
    public boolean equals(@androidx.annotation.Nullable Object obj) {
        Post post = (Post) obj;
        return details.matches(post.getDetails());
    }
}
~~~

Every post in the list will be represented by this model and two posts will be considered same if their details are same.
Following is the PostAdapter class :

~~~ java
public class PostAdapter extends RecyclerView.Adapter<PostAdapter.Holder> {
ArrayList<Post> data=new ArrayList<>();
LayoutInflater inflater;
Context context;
int mode =0;

public PostAdapter(Context context){
    this.context=context;
    inflater=LayoutInflater.from(context);
}

    @NonNull
    @Override
    public Holder onCreateViewHolder(@NonNull ViewGroup viewGroup, int i) {
    View view = null;
         view = inflater.inflate(R.layout.item_post, viewGroup, false);
        final Holder holder = new Holder(view);
        return holder;
    }

    @Override
    public void onBindViewHolder(@NonNull Holder holder, int i) {
        holder.topic.setText(data.get(i).getTopic());
        holder.details.setText(data.get(i).getDetails());
        Picasso.get().load(data.get(i).getImageURL()).into(holder.postImage);
    }



    @Override
    public int getItemCount() {
        return data.size();
    }


    class Holder extends RecyclerView.ViewHolder{
        TextView topic, details;
        ImageView postImage;

        public Holder(@NonNull View itemView) {
            super(itemView);
            topic=itemView.findViewById(R.id.topic);
            details=itemView.findViewById(R.id.details);
            postImage=itemView.findViewById(R.id.post_image);
        }
    }
   public void setData(ArrayList<News> data){
        this.data=data;
    }

    public void setMode(int mode) {
        this.mode = mode;
    }
    public Date getLastItemDate() {
        return data.get(data.size()-1).getDate();
    }
  }
~~~

The method getLastItemDate() will help us locate the last post in the list of posts.
Now we'll add mPosts number of posts firstly.

~~~java
void addNews(int mPosts){
    Query ref = FirebaseDatabase.getInstance().getReference()
            .child("database")
            .child("posts")
            .orderByChild("date")
            .limitToFirst(mPosts);
    ref.addChildEventListener(new ChildEventListener() {
        @Override
        public void onChildAdded(@NonNull DataSnapshot dataSnapshot, @Nullable String s) {
            Post post = dataSnapshot.getValue(Post.class);
            posts.add(post);
            postAdapter.setData(posts);
            postList.setHasFixedSize(true);
            postList.setLayoutManager(layoutManager);
            postList.setAdapter(postAdapter);



        }

        @Override
        public void onChildChanged(@NonNull DataSnapshot dataSnapshot, @Nullable String s) {

        }

        @Override
        public void onChildRemoved(@NonNull DataSnapshot dataSnapshot) {

        }

        @Override
        public void onChildMoved(@NonNull DataSnapshot dataSnapshot, @Nullable String s) {

        }

        @Override
        public void onCancelled(@NonNull DatabaseError databaseError) {

        }
    });
}
~~~

Now in the onCreate(), call this function and override the onScrolled of the RecyclerView to load more data subsequently.



~~~ java
protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
       postList=view.findViewById(R.id.post_list);
       layoutManager = new LinearLayoutManager(getContext());
       postAdapter = new PostAdapter(getContext());

       addNews(mPosts);
       postList.addOnScrollListener(new RecyclerView.OnScrollListener() {
           @Override
           public void onScrolled(@NonNull RecyclerView recyclerView, int dx, int dy) {
               super.onScrolled(recyclerView, dx, dy);
               int id = layoutManager.findLastCompletelyVisibleItemPosition();
               if(id>=mPosts-1){
                   addNewPost(postAdapter.getLastItemDate());
               }
           }
       });
}
~~~

Whenever the list is scrolled a new Post is added by the following method.
~~~java
void addNewPost(String id){
        Query ref = FirebaseDatabase.getInstance().getReference()
                .child("database")
                .child("post")
                .orderByChild("date")
                .startAt(id)
                .limitToFirst(1);
      ref.addChildEventListener(new ChildEventListener() {
          @Override
          public void onChildAdded(@NonNull DataSnapshot dataSnapshot, @Nullable String s) {
              if (dataSnapshot.exists()) {
                  Post post = dataSnapshot.getValue(Post.class);

                  if(!posts.contains(post)) {
                      posts.add(post);
                      postAdapter.setData(posts);
                      postAdapter.notifyDataSetChanged();
                  }


              }
          }

          @Override
          public void onChildChanged(@NonNull DataSnapshot dataSnapshot, @Nullable String s) {

          }

          @Override
          public void onChildRemoved(@NonNull DataSnapshot dataSnapshot) {

          }

          @Override
          public void onChildMoved(@NonNull DataSnapshot dataSnapshot, @Nullable String s) {

          }

          @Override
          public void onCancelled(@NonNull DatabaseError databaseError) {

          }
      });

    }
~~~

This is my implementation of pagination in Firebase. This maintains unique posts while loading more data on scrolling the list of posts.
