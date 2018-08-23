<template>
  <div id="app">
    <h1> {{ title }} </h1>
    <h3>New Book</h3>
    <form v-on:submit.prevent="onSubmit">
      <div>
        <input name="title" type="text" placeholder="title" v-model="book.title" />
      </div>
      <div>
        <input name="year" type="text" placeholder="year" v-model="book.year" />
      </div>
      <div>
        <input name="author" type="text" placeholder="author" v-model="book.author" />
      </div>
      <div>
        <label for="read">Read?</label>
        <input type="checkbox" v-model="book.read" id="read" name="read" />
      </div>
      <button v-if="updating">Update</button>
      <button v-else>Add</button>
    </form>

    <h3>All Books</h3>
    <table>
      <tr>
        <th>Title</th>
        <th>Year</th>
        <th>Author</th>
        <th>Read</th>
        <td>Update</td>
        <td>Delete</td>
      </tr>
      <tr v-for="(b, index) in books" :key="b.title">
        <td>{{ b.title }}</td>
        <td>{{ b.year }}</td>
        <td>{{ b.author }}</td>
        <td v-if="b.read">✓</td>
        <td v-else> </td>
        <td v-on:click.prevent="onEdit(index)"><a>✎</a></td>
        <td v-on:click.prevent="onDelete(index)"><a>✗</a></td>
      </tr>
    </table>
  </div>
</template>

<script>

import * as ds from 'deepstream.io-client-js';

export default {
  name: 'App',
  data () {
    return {
      ds: ds('localhost:6020'),
      books$$: null,
      title: 'My Books Manager',
      updating: false,
      updateIndex: 0,
      books: [],
      book: {
        title: '',
        year: '',
        author: '',
        read: false
      }
    };
  },
  created () {
    this.ds.login({}, () => {
      console.log('logged in');
    });

    this.books$$ = this.ds.record.getList('books');

    // Entry added
    this.books$$.on('entry-added', (recordName, index) => {
      this.ds.record.getRecord(recordName).whenReady(record => {
        record.subscribe(data => {
          if (!data.id) {
            if (data.title) {
              data.id = record.name;
              this.books.push(data);
            }
          } else {
            this.books = this.books.map(b => {
              if (data.id === b.id) {
                b = data;
              }
              console.log(b);
              return b;
            });
          }
        }, true);
      });
    });

    this.books$$.on('entry-removed', (recordName, index) => {
      this.ds.record.getRecord(recordName).whenReady(record => {
        record.subscribe(data => {
          this.books.splice(this.books.indexOf(data, 1));
        }, true);
      });
    });
  },
  methods: {
    onSubmit () {
      if (this.updating) {
        this.onUpdate();
        return;
      }
      this.books.push(this.book);
      this.book = {
        title: '',
        year: '',
        author: '',
        read: false
      };
    },
    onEdit (index) {
      this.updating = true;
      this.updateIndex = index;
      this.book = this.books[index];
    },
    onDelete (index) {
      this.books.splice(index, 1);
    },
    onUpdate () {
      const recordName = this.books[this.updateIndex].id;
      const bookRecord = this.ds.record.getRecord(recordName);
      bookRecord.set(this.book);
      this.book = {
        title: '',
        year: '',
        author: '',
        read: false
      };
      this.updating = false;
    }
  }
};
</script>

<style>
#app {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px auto 0;
}
h1, h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
table {
  margin: 0 auto;
}
td {
  padding: 0.1em 0.9em;
}
form > div {
  margin-bottom: 0.5em;
}
input {
  padding: 0.25em;
}
button {
  padding: 0.25em 1em;
  background-color: rgb(47, 109, 145);
  color: white;
  border: 1px solid rgb(47, 109, 145);
}
</style>
