<template>
  <div id="app" class="text-center container-fluid">
    <h1> {{ title }} </h1>
    <h3>New Album</h3>
    <form v-on:submit.prevent="onSubmit" class="form">
      <div class="row justify-content-center">
        <div class="col-md-4 col-sm-6">
          <div class="form-group">
            <input name="title" type="text" class="form-control" placeholder="title" v-model="album.title" />
          </div>
          <div class="form-group">
            <input name="year" type="text" class="form-control" placeholder="year" v-model="album.year" />
          </div>
          <div class="form-group">
            <input name="artist" type="text" class="form-control" placeholder="artist" v-model="album.artist" />
          </div>
        </div>
      </div>
      <!-- <div>
        <label for="read">Read?</label>
        <input type="checkbox" v-model="album.read" id="read" name="read" />
      </div> -->
      <button class="btn btn-primary" v-if="updating">Update</button>
      <button class="btn btn-primary" v-else>Add</button>
    </form>
    <hr>
    <h3>All Albums</h3>
    <div class="row justify-content-center">
      <div class="col-md-10">
        <table class="table table-dark table-sm table-hover">
          <thead>
            <tr>
              <th>Title</th>
              <th>Year</th>
              <th>Artist</th>
              <!-- <th>Read</th> -->
              <th></th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(a, index) in albums" :key="a.title">
              <td>{{ a.title }}</td>
              <td>{{ a.year }}</td>
              <td>{{ a.artist }}</td>
              <!-- <td v-if="a.read">✓</td> -->
              <!-- <td v-else> </td> -->
              <td v-on:click.prevent="onEdit(index)">
                <a class="btn btn-sm btn-primary">✎</a>
                <a class="btn btn-sm btn-danger">✗</a>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script>

import './assets/main.css';
import * as ds from 'deepstream.io-client-js';

export default {
  name: 'App',
  data () {
    return {
      ds: ds('localhost:6020'),
      albums$$: null,
      title: 'My Albums Manager',
      updating: false,
      updateIndex: 0,
      albums: [],
      album: {
        title: '',
        year: '',
        artist: ''
      }
    };
  },
  created () {
    this.ds.login({}, () => {
      console.log('logged in');
    });

    this.albums$$ = this.ds.record.getList('albums');
    this.albums$$.whenReady((list) => {
      list.getEntries().forEach(entry => {
        this.ds.record.getRecord(entry).whenReady(album => {
          this.albums.push(album.get());
        });
      });
    });

    // Entry added
    this.albums$$.on('entry-added', (recordName, index) => {
      this.ds.record.getRecord(recordName).whenReady(record => {
        record.subscribe(data => {
          if (!data.id) {
            if (data.title) {
              data.id = record.name;
              this.albums.push(data);
            }
          } else {
            this.albums = this.albums.map(b => {
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

    this.albums$$.on('entry-removed', (recordName, index) => {
      this.ds.record.getRecord(recordName).whenReady(record => {
        record.subscribe(data => {
          this.albums.splice(this.albums.indexOf(data, 1));
        }, true);
      });
    });
  },
  methods: {
    onSubmit () {
      const recordName = this.album.id || 'album/' + this.ds.getUid();

      this.ds.record.has(recordName, (err, has) => {
        if (err) {
          return;
        }
        if (has) {
          this.onUpdate();
        } else {
          const albumRecord = this.ds.record.getRecord(recordName);
          albumRecord.set(this.album);

          this.albums$$.addEntry(recordName);

          this.album = {
            title: '',
            year: '',
            artist: ''
          };
        }
      });
    },
    onEdit (index) {
      this.updating = true;
      this.updateIndex = index;
      this.album = this.albums[index];
    },
    onDelete (index) {
      this.albums$$.removeEntry(this.albums[index].id);
    },
    onUpdate () {
      const recordName = this.albums[this.updateIndex].id;
      const albumRecord = this.ds.record.getRecord(recordName);
      albumRecord.set(this.album);
      this.album = {
        title: '',
        year: '',
        artist: '',
        read: false
      };
      this.updating = false;
    }
  }
};
</script>

<style>
</style>
