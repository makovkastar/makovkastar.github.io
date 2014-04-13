---
layout: post
title: "Android AutoCompleteTextView with suggestions from a web service"
date: 2014-04-12 09:52:04 +0300
comments: true
categories: android, AutoCompleteTextView
---

# Android AutoCompleteTextView with suggestions from a web service #

In the last update for my Android project [BookTracker](https://play.google.com/store/apps/details?id=com.melnykov.booktracker) I have implemented an [AutoCompleteTextView](http://developer.android.com/reference/android/widget/AutoCompleteTextView.html) with suggestions for a book title which are fetched from the [Google Books](https://developers.google.com/books/). There were a few requirements for such a view:

- Suggestions data fetching must be performed in a separate thread
- Data fetching should start only when a user pauses typing (to prevent a bunch of requests to a server after every entered character)
- Suggestions should be shown only if the user enters a string of some minimum length (no reason to start data fetching for string of 2 or 3 characters) 
- An animated progress bar must be shown on the right side of the view when suggestions fetching is in a progress

The final result looks like this:

{% img /images/bloggif_53483ba352390.gif %}

## Step 1 - Implement an adapter for AutoCompleteTextView

An adapter for the `AutoCompleteTextView` is a core component where suggestions are loaded and stored. The `BookAutoCompleteAdapter` must implement the [Filterable](http://developer.android.com/reference/android/widget/Filterable.html) interface in order for you to capture the user input from the `AutoCompleteTextView` and pass it as a search criteria to the web service. A single method of the `Filterable` interface is `getFilter()` that must return a [Filter](http://developer.android.com/reference/android/widget/Filter.html) instance which is actually performs the data loading and publishing. `Filter` subclasses must implement 2 methods: `performFiltering (CharSequence constraint)` and `publishResults (CharSequence constraint, Filter.FilterResults results)`. 

The `performFiltering` method is invoked in a worker thread so no need to create and start a new thread manually. It's done already by the `Filter` itself. The `publishResults` method is invoked in the UI thread to publish the filtering results in the user interface.


##### BookAutoCompleteAdapter.java:

```java
public class BookAutoCompleteAdapter extends BaseAdapter implements Filterable {

    private static final int MAX_RESULTS = 10;
    private Context mContext;
    private List<Book> resultList;

    public BookAutoCompleteAdapter(Context context) {
        mContext = context;
    }

    @Override
    public int getCount() {
        return resultList.size();
    }

    @Override
    public Book getItem(int index) {
        return resultList.get(index);
    }

    @Override
    public long getItemId(int position) {
        return position;
    }

    @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        if (convertView == null) {
            LayoutInflater inflater = (LayoutInflater) mContext
                    .getSystemService(Context.LAYOUT_INFLATER_SERVICE);
            convertView = inflater.inflate(R.layout.simple_dropdown_item_2line, parent, false);
        }
        ((TextView) convertView.findViewById(R.id.text1)).setText(getItem(position).getTitle());
        ((TextView) convertView.findViewById(R.id.text2)).setText(getItem(position).getAuthor());
        return convertView;
    }

    @Override
    public Filter getFilter() {
        Filter filter = new Filter() {
            @Override
            protected FilterResults performFiltering(CharSequence constraint) {
                FilterResults filterResults = new FilterResults();
                if (constraint != null) {
                    resultList = findBooks(mContext, constraint.toString());

                    // Assign the data to the FilterResults
                    filterResults.values = resultList;
                    filterResults.count = resultList.size();
                }
                return filterResults;
            }

            @Override
            protected void publishResults(CharSequence constraint, FilterResults results) {
                if (results != null && results.count > 0) {
                    notifyDataSetChanged();
                }
                else {
                    notifyDataSetInvalidated();
                }
            }};
        return filter;
    }

    private List<Book> findBooks(Context context, String bookTitle) {
        GoogleBooksProtocol protocol = new GoogleBooksProtocol(context, MAX_RESULTS);
        return protocol.findBooks(bookTitle, null);
    }
}
```


## Step 2 - Create an XML layout for a suggestion list row

After suggestions are fetched, a list of results is displayed bellow the view. Each list row consists of two lines: a book name and an author.

#### simple\_dropdown\_item_2line.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<TwoLineListItem xmlns:android="http://schemas.android.com/apk/res/android"
                 android:layout_width="match_parent"
                 android:layout_height="wrap_content"
                 android:minHeight="?android:attr/listPreferredItemHeight"
                 android:mode="twoLine"
                 android:paddingStart="?android:attr/listPreferredItemPaddingStart"
                 android:paddingEnd="?android:attr/listPreferredItemPaddingEnd">

    <TextView android:id="@+id/text1"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:layout_marginTop="@dimen/margin_default"
              android:textAppearance="?android:attr/textAppearanceLargePopupMenu"/>

    <TextView android:id="@+id/text2"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:layout_below="@id/text1"
              android:layout_alignStart="@id/text1"
              android:layout_marginBottom="@dimen/margin_default"
              android:textAppearance="?android:attr/textAppearanceSmall"/>

</TwoLineListItem>

```

## Step 3 - Add a delay before sending a data request to a web service

With a standard `AutoCompleteTextView` a filtering will be initiated after each entered character. If the user is typing a text nonstop, data fetched for the previous request may become invalid on every new letter appended to the search string. You get extra expensive and unnecessary network calls, chance of exceeding API limits of your web service, stale suggestion results loaded for an incomplete search string. The way we go - add a small delay before user types the character and the request is sent to the web. If during this time the user enters the next character - the request for the previous search string is cancelled and rescheduled for the delay time again. If the user doesn't change the text during the delay time - the request is sent. To implement this behaviour we create a custom implementation of `AutoCompleteTextView` and override the method `performFiltering(CharSequence text, int keyCode)`. The variable `mAutoCompleteDelay` defines time in milliseconds after the request will be sent to a server if user didn't change the search string.

#### DelayAutoCompleteTextView.java

```java
public class DelayAutoCompleteTextView extends AutoCompleteTextView {

    private static final int MESSAGE_TEXT_CHANGED = 100;
    private static final int DEFAULT_AUTOCOMPLETE_DELAY = 750;

    private int mAutoCompleteDelay = DEFAULT_AUTOCOMPLETE_DELAY;
    private ProgressBar mLoadingIndicator;

    private final Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            DelayAutoCompleteTextView.super.performFiltering((CharSequence) msg.obj, msg.arg1);
        }
    };

    public DelayAutoCompleteTextView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public void setLoadingIndicator(ProgressBar progressBar) {
        mLoadingIndicator = progressBar;
    }

    public void setAutoCompleteDelay(int autoCompleteDelay) {
        mAutoCompleteDelay = autoCompleteDelay;
    }

    @Override
    protected void performFiltering(CharSequence text, int keyCode) {
        if (mLoadingIndicator != null) {
            mLoadingIndicator.setVisibility(View.VISIBLE);
        }
        mHandler.removeMessages(MESSAGE_TEXT_CHANGED);
        mHandler.sendMessageDelayed(mHandler.obtainMessage(MESSAGE_TEXT_CHANGED, text), mAutoCompleteDelay);
    }

    @Override
    public void onFilterComplete(int count) {
        if (mLoadingIndicator != null) {
            mLoadingIndicator.setVisibility(View.GONE);
        }
        super.onFilterComplete(count);
    }
}
```

## Step 4 - Add an animated progress to the view

It's very important to provide a feedback to the user when he is typing the text. We have to display an animated progress in the same view  to say to the user "Hey, suggestions are loading right now and will be displayed shortly". In that way user will expect something to happen and can wait until response received. Without this feedback the user will even not suspect that a field has suggestions. 

We put the `ProgressBar` widget and `DelayAutoCompleteTextView` to the `FrameLayout` and align the progress to the right side of the input field. We set `android:visibility="gone"` as the initial state of the progress:



```xml
<FrameLayout android:layout_width="match_parent"
                 android:layout_height="wrap_content"
                 android:layout_margin="@dimen/margin_default">

    <com.melnykov.booktracker.ui.DelayAutoCompleteTextView
            android:id="@+id/et_book_title"
            android:inputType="textCapSentences"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:paddingRight="@dimen/padding_auto_complete"
            android:imeOptions="flagNoExtractUi|actionSearch"
            android:hint="@string/hint_book_title"/>

    <ProgressBar
            android:id="@+id/pb_loading_indicator"
            style="?android:attr/progressBarStyleSmall"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center_vertical|right"
            android:layout_marginRight="@dimen/margin_default"
            android:visibility="gone"/>
</FrameLayout>
```

The `ProgressBar` is connected to the `DelayAutoCompleteTextView` via `setLoadingIndicator(ProgressBar view)` method of the latter. It's visibility will be set to `View.GONE` when a filtering starts and to `View.GONE` when completed. 

Now place this layout inside where you need your  

## Step 5 - Assemble components together

Now when we have all components ready we can assemble them together: 

```java
DelayAutoCompleteTextView bookTitle = (DelayAutoCompleteTextView) findViewById(R.id.et_book_title);
    bookTitle.setThreshold(THRESHOLD);
    bookTitle.setAdapter(new BookAutoCompleteAdapter(this)); // this is Activity instance
    bookTitle.setLoadingIndicator(
                (android.widget.ProgressBar) findViewById(R.id.pb_loading_indicator));
    bookTitle.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> adapterView, View view, int position, long id) {
                Book book = (Book) adapterView.getItemAtPosition(position);
                bookTitle.setText(book.getTitle());
            }
        });
```

` bookTitle.setThreshold(THRESHOLD)` specifies the minimum number of characters the user has to type in the edit box before the drop down list is shown.

`bookTitle.setLoadingIndicator((android.widget.ProgressBar) findViewById(R.id.pb_loading_indicator))` binds the `ProgressBar` view to the `DelayAutoCompleteTextView`.

It's important to set `OnItemClickListener` for the `DelayAutoCompleteTextView` and set a correct value to the target input field. Without doing that a string obtained via call to `Object.toString()` will be pasted to the field. 
