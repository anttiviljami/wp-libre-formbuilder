<template>
  <div class="wplfb-sandbox">
    <div class="wplfb-sandbox__formselect">
      <select name="formselect" @change="() => changeForm()">
        <option v-for="form in store.state.forms" :value="form.form.ID">
          {{ form.form.post_name }}
        </option>
        <option value="0">New</option>
      </select>
    </div>
    <draggable
      class="wplfb-sandbox__playfield"
      :options="{group: {name: 'fields', put: true }}"
      @move="moveHandler"
    >
      <p>Drag fields into me!</p>
      <template v-for="(value, key) in store.state.tree">
        <!-- <header>{{ key }}</header> -->
        <!-- <pre>{{ value.attributes }}</pre> -->
        <wplfb-field
          :name="key"
          :fieldData="value"
          :element="value.element"
          :attributes="value.attributes"
          :takesChildren="value.takesChildren"
          :removeHandler="removeHandler"
          :moveHandler="moveHandler"
          :startHandler="startHandler"
          :endHandler="endHandler"
          :cloneHandler="cloneHandler"
        >
          <!-- Children will be placed here -->
        </wplfb-field>
      </template>
    </draggable>

    <draggable
      class="wplfb-sandbox__tools"
      :options="{group: {name: 'fields', pull: 'clone', put: false }}"
      @move="moveHandler"
      @end="endHandler"
      @clone="cloneHandler"
    >
      <template v-for="(value, key) in fields">
        <!-- <header>{{ key }}</header> -->
        <!-- <pre>{{ value.attributes }}</pre> -->
        <wplfb-field
          :name="key"
          :fieldData="value"
          :element="value.element"
          :attributes="value.attributes"
          :takesChildren="value.takesChildren"
          :removeHandler="removeHandler"
          :moveHandler="moveHandler"
          :startHandler="startHandler"
          :endHandler="endHandler"
          :cloneHandler="cloneHandler"
        >
          <!-- Children will be placed here -->
        </wplfb-field>
      </template>
    </draggable>

    <wplfb-notification v-if="store.state.notification.show" :msg="store.state.notification.text" :type="store.state.notification.type" />
  </div>
</template>

<script>
import field from './Field';
import notification from './Notification';
import draggable from 'vuedraggable';

let base;
if (window.location.port !== 80 || window.location.port !== 443) {
  base = window.REST_URL; // Defined in index.html
} else {
  base = '';
}

const store = {
  debug: true,
  state: {
    form_id: undefined,
    tree: {

    },
    forms: [

    ],
    currentMove: {

    },
    notification: {
      show: false,
      text: undefined,
      type: undefined,
    },
  },

  getTree() {
    return this.state.tree;
  },

  addForm(form, cb) {
    this.debug && console.log('Adding form', form);
    this.state.forms.push(form);

    cb && cb(form);
  },

  updateFormID(id, cb) {
    this.debug && console.log('Update form_id', id);
    this.state.form_id = id;

    cb && cb(id);
  },

  updateTree(newTree, cb) {
    this.debug && console.log('Update tree', newTree);
    this.state.tree = newTree;

    cb && cb(newTree);
  },

  updateMove(fieldData, cb) {
    this.debug && console.log('Update move', fieldData);
    this.state.currentMove = fieldData;

    cb && cb(fieldData);
  },

  showNotification(msg, type) {
    this.debug && console.log('Show notification', msg);

    this.state.notification.show = true;
    this.state.notification.text = msg;
    this.state.notification.type = 'success';

    setTimeout(() => {
      this.state.notification.show = false;
    }, 2500);
  }
};

const saveForm = () => {
  console.log('does this run');
  fetch(`${base}/wp-json/wplfb/forms/form`, {
    method: 'POST',
    body: JSON.stringify(store.getTree()),
    headers: {
      'Content-Type': 'application/json',
    },
  })
    .then(r => r.json())
    .then(r => {
      console.log(r);
      if (!r.error) {
        this.notifyOfSuccess(r.success);
      } else {
        this.notifyOfError(r.error);
      }
    })
    .catch(err => {
      console.log(err);
      this.notifyOfError(err);
    });
};

export default {
  name: 'builder',
  components: {
    draggable,
    'wplfb-field': field,
    'wplfb-notification': notification,
  },

  beforeMount() {
    fetch(`${base}/wp-json/wplfb/forms/forms`)
      .then(r => r.json())
      .then(r => {
        r.forEach(store.addForm.bind(store));
      })
      .catch(err => {
        throw err;
      });

    fetch(`${base}/wp-json/wplfb/fields`)
      .then(r => r.json())
      .then(r => {
        Object.keys(r.fields).forEach(key => {
          const value = r.fields[key];

          console.log(key, value);
          this.$set(this.fields, value.name, {
            attributes: value.dom.attributes,
            element: value.dom.element,
            takesChildren: value.takesChildren,
          });
        });
      })
      .catch(err => {
        throw err;
      });

    setTimeout(() => {
      store.showNotification('test', 'success');
    }, 2500);
  },

  methods: {
    preventDrag(e) {
      console.log(e);
      return false;
    },

    removeHandler() {
      console.log('Remove!');
    },

    startHandler() {
      store.updateMove(this.fieldData);
      console.log('Start!');
    },

    moveHandler(event) {
      const { draggedContext, relatedContext } = event;

      console.log('Move!', draggedContext, relatedContext);
    },

    endHandler(event) {
      console.log('end');
      console.log(event);

      const newTree = {};
      store.updateTree(newTree, saveForm);
    },

    cloneHandler() {
      console.log(this);
      console.log('Clone!');
    },

    changeHandler(added, removed, moved) {

    },

    changeForm() {
      const select = this.$el.querySelector('select[name="formselect"]');
      const value = parseInt(select.value, 10);

      if (value !== 0) {
        store.updateFormID(value);
        fetch(`${base}/wp-json/wplfb/forms/form?form_id=${value}`)
          .then(r => r.json())
          .then(r => {
            const { fields } = r;
            store.updateTree(JSON.parse(fields));
          })
          .catch(err => {
            throw err;
          });
      }
    },
  },
  data () {
    return {
      fields: {
        /* 'Text': {
          element: 'input',
          attributes: {
            type: 'text',
            placeholder: 'Test',
            class: 'test',
            name: 'test',
          },
          takesChildren: false
        },

        'Wrapper': {
          element: 'div',
          attributes: {
            class: ['b', 'c'],
          },
          takesChildren: true
        } */
      },

      store,
    };
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="stylus">
.wplfb-sandbox {
  text-align: left
  display: flex
  flex-flow: row nowrap
  align-items: flex-start
  justify-content: space-between

  &__playfield {
    width: 65%
    min-height: 100vh
  }

  &__tools {
    width: 30%

    .wplfb-field {
      .remove {
        display: none
      }
    }
  }

  .wplfb-field {
    margin-bottom: 1rem

    &:last-child {
      margin-bottom: 0
    }
  }
}

</style>
