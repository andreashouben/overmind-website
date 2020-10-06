# Managing lists

Why do we even have a guide to managing lists? Well, lists are a type of state that differ from other types of state. Both from the perspective of the state itself, but also transforming that state into UI.

{% hint style="info" %}
The discussion of lists is not specific to Overmind and there are no limitations on how you want to approach this. This guide is rather to help you think about how you structure entities \(data with an id\) to optimally access and render them.
{% endhint %}

## Defining the state

When we want to render a list of something we want an **array**. This is the data structure we instinctively go to as it basically is a list. But arrays are rarely the way you want to store the actual data of the list.

If your list consists of posts, these posts are most likely entities from the server which are unique, and have a unique id. A data structure that better manages uniqueness is an **object**.[EDIT ON GITHUB](https://github.com/cerebral/overmind/edit/next/packages/overmind-website/examples/guide/managinglists/object.ts)

```typescript
{
  "uniqueId1": {},
  "uniqueId2": {},
  "uniqueId3": {}
}
```

Another benefit from using an object is that you have a direct reference to the entity by using its id. No need to iterate an array to find what you are looking for. So this is a good rule of thumb: if the state you are defining has unique identifiers they should most likely be stored as an object, not an array.

But we still want to use an array when we transform the state into a UI. Let us see what we can do about that.

## Derive to a list

In Overmind it is encouraged that you derive these dictionaries of entities to a list by deriving the state. The most simple way to do this is:

{% tabs %}
{% tab title="overmind/state.js" %}
```typescript
import { derived } from 'overmind';

export const state = {
  posts: {}
  postsList: derived(state => Object.values(state.posts))
}
```
{% endtab %}
{% endtabs %}

Now when we point to **state.postsList** we get an array of posts. What is important to remember here is that we do not create the list again whenever we point to this state value. It is only run again if there is a change to the referenced posts’ state.

## Sorting

Now we have optimally stored our posts in a dictionary for easy access by id. We have also created a derived state which converts this dictionary to an array whenever the source dictionary changes. Though most likely you want to sort the list. Typically lists are sorted chronologically and our posts item has a **datetime** field.

{% tabs %}
{% tab title="overmind/state.js" %}
```typescript
export const state = {
  posts: {}
  postsList: state =>
    Object.values(state.posts)
      .sort((postA, postB) => {
        if (postA.datetime > postB.datetime) {
          return 1
        } else if (postA.datetime < postB.datetime) {
          return -1
        }

        return 0
      })
}
```
{% endtab %}
{% endtabs %}

There we go, our posts are now shown chronologically. But maybe we only want to show some of the posts that are available? Maybe the list should only contain the ten latest entries?

## Filtering

To limit the number of posts shown we can create a new state and use it inside our derived state to limit the number of results.

{% tabs %}
{% tab title="overmind/state.ts" %}
```typescript
import { Derive } from 'overmind'

export type Post {
  id: string
  title: string
  body: string
  datetime: number
}

export type State = {
  posts: { [id: string] : Post }
  showCount: number
  postsList: Derive<State, Post[]>
}

export const state: State = {
  posts: {},
  showCount: 10,
  postsList: state =>
    Object.values(state.posts)
      .sort((postA, postB) => {
        if (postA.datetime > postB.datetime) {
          return 1
        } else if (postA.datetime < postB.datetime) {
          return -1
        }

        return 0
      })
      .slice(0, state.showCount)
}
```
{% endtab %}
{% endtabs %}

Now if we change the **showCount** state our derived list will indeed update.

## Rendering lists

So now let’s look at how we would consume such a list. Let us look at a straightforward example first:

{% tabs %}
{% tab title="React" %}
{% code title="components/App.tsx" %}
```typescript
import * as React from 'react'
import { useOvermind } from '../overmind'

const Posts: React.FunctionComponent = () => {
  const { state } = useOvermind()

  return (
    <ul>
      {state.postsList.map(post => 
        <li key={post.id}>{post.title}</li>
      )}
    </ul>
  )
}

export default App
```
{% endcode %}
{% endtab %}

{% tab title="Angular" %}
{% code title="components/posts.component.ts" %}
```typescript
import { Component } from '@angular/core';
import { connect } from '../overmind'

@Component({
  selector: 'app-posts',
  template: `
  <ul>
    <li *ngFor="let post of overmind.state.postsList;trackby: trackById">
      {{post.title}}
    </li>
  </ul>
  `
})
@connect()
export class List {
  trackById(index, post) {
    return post.id
  }
}
```
{% endcode %}
{% endtab %}

{% tab title="Vue" %}
{% code title="components/Posts.vue" %}
```typescript
<template>
  <ul>
    <li v-for="post in state.postsList" :key="post.id>
      {{ post.title }}
    </li>
  </ul>
</template>
```
{% endcode %}
{% endtab %}
{% endtabs %}

Now this approach will work perfectly fine. The component will render the list and update it whenever it needs to. The only drawback with this approach is that any change to individual posts will also cause the component to render, specifically the **title** of a post since that is the only thing we are looking at. This is because this one component is looking at all the posts. We can optimize this by passing each post down to a child component:

{% tabs %}
{% tab title="React" %}
{% code title="components/Post.tsx" %}
```typescript
import * as React from 'react'
import { useOvermind } from '../overmind'
import { Post as TPost } from '../overmind/state'

type Props = {
  post: TPost
}

const Post: React.FunctionComponent<Props> = ({ post }) => {
  // We still need to use the hook so that the component tracks
  // changes to the post
  useOvermind()

  return (
    <li>{post.title}</li>
  )
}

export default Post
```
{% endcode %}
{% endtab %}

{% tab title="Angular" %}
{% code title="components/post.component.ts" %}
```typescript
import { Component, Input, ChangeDetectionStrategy } from '@angular/core';
import { Post } from '../overmind/state'

@Component({
  selector: 'app-post',
  template: `
  <li *track>
    {{post.title}}
  </li>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class List {
  @Input() post: Post;
}
```
{% endcode %}
{% endtab %}

{% tab title="Vue" %}
{% code title="components/Post.vue" %}
```typescript
<template>
  <li>{{ post.title }}</li>
</template>
<script>
export {
  props: ['post']
}
</script>
```
{% endcode %}
{% endtab %}
{% endtabs %}

{% tabs %}
{% tab title="React" %}
{% code title="components/Posts.tsx" %}
```typescript
import * as React from 'react'
import { useOvermind } from '../overmind'
import Post from './Post'

const Posts: React.FunctionComponent = () => {
  const { state } = useOvermind()

  return (
    <ul>
      {state.postsList.map(post => 
        <Post key={post.id} post={post} />
      )}
    </ul>
  )
}

export default Posts
```
{% endcode %}
{% endtab %}

{% tab title="Angular" %}
{% code title="components/posts.component.ts" %}
```typescript
import { Component } from '@angular/core';
import { Store } from '../overmind'

@Component({
  selector: 'app-posts',
  template: `
  <ul>
    <app-post
      *ngFor="let post of state.postsList;trackby: trackById"
      [post]="post"
    ></app-post>
  </ul>
  `
})
export class List {
  state = this.store.select()
  constructor(private store: Store) {}
}
```
{% endcode %}
{% endtab %}

{% tab title="Vue" %}
{% code title="components/Posts.vue" %}
```typescript
<template>
  <ul>
    <post-component v-for="post in state.postsList" :post="post" :key="post.id"></post-component>
  </ul>
</template>
<script>
import PostComponent from './Post'

export default {
  name: 'Posts',
  components: {
    PostComponent,
  },
}
</script>
```
{% endcode %}
{% endtab %}
{% endtabs %}

Now the **Posts** component only cares about changes to the list itself, while each **Post** component cares about its corresponding post title. It means that if the title of a post updates, only the component that actually cares about that post renders again. If the list itself changes only the **Posts** component will render.

### Summary <a id="managing-lists-summary"></a>

Managing lists has two considerations: defining how to store the data of the list, and how to actually render the list. It can be a good idea to store data entities with unique ids as a dictionary and rather use a **derived** state to produce the array itself. This gives you best of both worlds – easy lookup using the id, and a cached list that only updates when dependent state updates.

