#swpMVC

swpMVC is a lightweight MVC framework built to bring some of the experience of other
rapid application development frameworks to WordPress. Inspired largely by Rails, Express
and FuelPHP, it aims to make routing, modeling, and rendering easy, giving you more control
over your code structure than WordPress gives out of the box, without adding too much
extra work.

##Tutorial

The quickest way to get familiar with the swpMVC framework is with the TodoApp tutorial found
[here](http://streetwise-media.github.com/swpmvc_todos/#looking-at-other-users-todos)

##Starter Plugin

***

While swpMVC currently does not include any code generation, you can get a jump start on plugin development by examining
(and using) the starter plugin found in the starter_plugin directory.

###Why a singleton?

I know a singleton is an upgraded global variable, but your plugins need to add WordPress filters and
actions. Using a singleton gives you easy access to the plugin class, and also makes sure you don't
end up running the methods you hang on your filters and actions more than once each.

###require_dependencies

This is called in the constructor, and this is where you should include your models and controllers. By
creating the plugin instance on the swp_mvc_init action, we ensure that the swpMVC core is loaded,
and the base classes which your models and controllers must extend will be available.

###add_actions

Also called in the constructor, this is used to hook the add_routes method to the swp_mvc_routes
filter, where you can add your plugin routes using the syntax described in the router section. By
placing this after the require_dependencies call in the constructor, you can be sure your controllers
are loaded when you add the routes to the system.

###add_routes

swpMVC follows a routing structure that more closely resembles Sinatra or Express than WordPress'
rewrite rules. See the next section for syntax.


##Router

***

### Adding routes

swpMVC routes are stored as an array of arrays, with each array stored representing one route using the following
structure

    <?php
    
        $route = array('controller' => 'ControllerClass',
                                'method' => 'ControllerMethod',
                                'route' => '/url/of/route/:p/:p'
                            );
        
There is no "automagic" routing, everything must be declared. This is done so that your routing structure is exactly as
you want, with no additional steps required to turn off magic routes.

### Routing parameters

Parameters in your route are represented with the token ":p"

They will be passed to your controller method in the same order they appear in the route. Skipping named parameters allows the
framework to use [only one additional querystring variable](http://codex.wordpress.org/Rewrite_API/add_rewrite_tag)

### Auto-flush rewrite rules

The core framework will monitor whether swpMVC routes have been added, modified or removed, and flush the rewrite
rules as needed, so there is no need to do this manually.

### Example

Here's a full example adding routes from your swpMVC plugin based off the example plugin included in the example
directory

    <?php
        
        public function add_routes($routes)
        {
            $r[] = array('controller' => 'swpMVC_Example_Controller',
                        'method' => 'wp_style',
                        'route' => '/recent_thumbs/wp_style');
            $r[] = array('controller' => 'swpMVC_Example_Controller',
                            'method' => 'swpmvc_style',
                            'route' => '/recent_thumbs/swpmvc_style');
            $r[] = array('controller' => 'swpMVC_Example_Controller',
                            'method' => 'render_post_form',
                            'route' => '/post_form/:p');
            $s =  array_merge($routes, $r);
            return $s;
        }


##Models

***

Models must extend the swpMVCBaseModel class. This class itself extends ActiveRecord\Model, from the
[PHP ActiveRecord library](http://phpactiverecord.org). For query syntax, CRUD operations, basic model definitions, and
overloading, refer to the ActiveRecord docs. The copy included in swpMVC [includes several modifications to make it more WordPress friendly, (the diff is backwards, sorry)](https://github.com/beezee/php-activerecord/compare/master...original)
, and the swpMVCBaseModel class includes several convenience methods, all of which are covered in this
section.

###public static function tablename

Instead of declaring the model table with a static variable, we use a static method. This allows us to do the following

    <?php
        
        public static function tablename()
        {
            global $wpdb;
            return $wpdb->prefix.'posts';
        }

This model would now be multisite aware. The advantage to using a method over a variable is that we can now dynamically
define the table property for our model.

###public static function conditions

This defines any conditions that should apply to every finder query that is generated by the model. For example if I wanted
to model draft posts only:
    
    <?php
        public static function conditions()
        {
            return array("post_status = ?", "draft");
        }
        
###public static function joins

This defines any joins that should apply to every finder query that is generated by the model. In general for related eager loading,
I favor include, using the joins method only when my conditions method relies on data in another table. An example of how this
can be used to model categories:

    <?php
    
    class Category extends swpMVCBaseModel
    {
        public static function tablename()
        {
            global $wpdb;
            return $wpdb->prefix.'terms';
        }
    
        public static function conditions()
        {
            global $wpdb;
            $tt = $wpdb->prefix.'term_taxonomy';
            return array("$tt.taxonomy = ?", 'category');
        }
        
        public static function joins()
        {
            global $wpdb;
            $t = self::tablename();
            $tt = $wpdb->prefix.'term_taxonomy';
            return "LEFT JOIN $tt ON $t.term_id = $tt.term_id";
        }
    }
    
Now any finder queries generated by the Category class will include a left join on the term_taxonomy table, and
filter results to include only those where the term_taxonomy.taxonomy field has a value of "category." Filtering
subsets with models is particularly relevant in WordPress, where different "types" of data are frequently lumped
together in single tables.

One catch to using the joins method, is that calls to the [models finder methods](http://www.phpactiverecord.org/projects/main/wiki/Finders)
will need to use table prefixes for any columns that are present in both the main and joined tables. For this I recommend your
Model::tablename() methods.


###Automatic stripslashes

Model properties in string format are automatically run through stripslashes when accessed directly. To override this, call the
properties method on an object, and access the properties from the resulting array.

###$model->render()

swpMVCBaseModel comes with an instance method 'render,' which accepts as an argument a Stamp view object (see [Views](/Streetwise-Wordpress-MVC/#views)
section for details,) and autopopulates the Stamp using the model properties.

###public function render_{{property_name}}

These methods act as overrides for your properties when called by the render method. For example, if a Stamp view object
passed to the render method contains a tag called post_name, and your Post model has a method called render_post_name,
the return value of that method will be used to populate the Stamp object in favor of the value of the instance property
post_name.

###public function needs_template_cleanup

This method accepts two parameters, property\_name and property\_value. You can overload this method in your models to
determine which values for a property will cause a Stamp tag of the format {{property\_name}}\_block to be stripped from the
template when rendered. Here's an example that will strip \_block tags for any falsy value, or a custom value if the property is
post\_title:

    <?php
        
        public function needs_template_cleanup($property_name, $property_value)
        {
            if ($property_name === 'post_title')
                return $property_value === false or
                    trim($property_value) === '' or
                    $property_value ===
                        'This post title strips the post_title_block tag';
            return $property_value === false or
                empty($property_value) or
                trim($property_value) === '';
        }
        
As shown in the example above, the {{property\_name}}_block tag is stripped when $model->needs\_template\_cleanup()
returns true. The default behavior is to check for falsy values.

###public function renderers

This method allows you to define additional renderers (not limited to model properties) that the model should be able to handle.
It must return an array where the keys are the Stamp tags to be replaced, and the value is an array where the first element is
the name of an instance method on the model which will return the value to be used in view population, and the second element
is an array of arguments, (an empty array if no arguments to be passed.)

For example:

    <?php
    
        public function renderers()
        {
            return array(
                'additional_render_tag' => array('additional_render_method', array())
            );
        }
        
This would replace a Stamp tag labeled 'additional_render_tag' with the result of calling $model->additional_render_method()
with no arguments.

###public static function controls

This method allows you to define form controls that should be used to interact with model properties, which will then be used by
the [render](/Streetwise-Wordpress-MVC/#models/model-render) method to populate any Stamp tags of the format control\_{{property name}} and additionally
control\_label\_{{property name}} if the label attribute is defined for the corresponding element in the returned array.
The return value from this method is again an associative array of arrays which must follow the structure below:

    <?php
    
        public static function controls()
        {
            return array(
                'property_name' => array(
                        'type' => 'input',
                        'label' => 'Property Name',
                        'input_type' => 'button'
                    )
            );
        }
        
The following must be true of any element in the controls array for it to be valid:

*   The key must match the model property that the control corresponds to.
*   type must be either input, select, or textarea
*   input_type is optional and only applies when type is set to input. Default is text.
*   label is optional. control\_label\_{{property name}} tags will not be replaced if no label property is defined
*   If type is select, an additional element is required with key 'options'. The value of this element must be an associative array of
    options for the dropdown, where key is the text for the option, and value is the value when that option is selected.
    
An example of a valid select control:

    <?php
    
        public function controls
        {
            return array(
                'property_name' => array(
                        'type' => 'select',
                        'label' => 'Property Name',
                        'options' => array(
                            'Option 1 Text' => 'option_1_value',
                            'Option 2 Text' => 'option_2_value'
                        )
                    )
            );
        }

When called by the $model->render() method, the generated controls will have their values set according to the values of the
model instance on which the render method was invoked, and the form will appear populated.

###Form prefixes

Each model rendering a form adds a prefix to its form elements. By default when invoked via $model->render(), the form prefix
will be the class name of the model instance invoking the render method. To override this, set the '\_prefix' property on the  model
instances form helper before calling render, as follows:

    <?php
        
        $model->form_helper()->_prefix = 'my_custom_form_prefix';
        $model->render($this->template('template_name'));
        
###Model::renderForm()

This method can be called statically on a model class to render an empty form for the class properties. The method accepts
two parameters. The first parameter is required and must be a valid Stamp view object to be populated, the second is an optional
form prefix (defaults to the class name.)

###A note about "through" relationships

Personally I've not had much success declaring [through relationships](http://www.phpactiverecord.org/projects/main/wiki/Associations#has_many_through)
with my activerecord models, so I have stuck to nesting includes during my finder calls manually. While the through relationship
does improve efficiency, I've decided it's not worth the trouble, as even without the generated joins you can typically get all
the data you will need for less than 10 queries with simple nested eager loading. Here's an example of getting 10 posts with their
related post tag and category data using simple nested eager loading:

    <?php
        
        $posts = Post::all(array('limit' => 10,
                'include' => array('postterms' =>
                    array('termtaxonomy' =>
                        array('term')
                    )
                )
            )
        );
        
If you do have some success modeling with the through relationships, please reach out via Github issues and I'll be happy to update
the docs, repo as needed.

###Included models


*       Post
*	PostMeta
*	TermRelationship
*	TermTaxonomy
*	Term
*       Category
*       Tag
*	Comment
*	User
*	UserMeta

All models can be found in models/wordpress_models.php. There's not alot of code, and the best way to learn model definition, as
well as see what added methods are available on each model is to view the source.

##Model meta

***

the swpMVCBaseModel class includes some methods for working with meta, a popular WordPress data structure. Any table with
columns foreign_key, meta_key, meta_value will work with these methods, once you've defined a
[$has_many relationship](http://www.phpactiverecord.org/projects/main/wiki/Associations) to a model with the name meta.

###$model->meta()

This method will return an empty array if there is no $has_many [relationship](http://www.phpactiverecord.org/projects/main/wiki/Associations)
named meta defined for the model on which it is called.

If the meta relationship is defined, it must point to a Model of a table with columns foreign_key, meta_key and meta_value.
By calling $model->meta() when these circumstances are met, the return value will be an associative array where keys are
equal to meta_key, and values are equal to the meta_value. In the case of duplicate meta_key rows for one model instance,
the meta_value will be an indexed array of all values found.

It is recommended to [eager load](http://www.phpactiverecord.org/projects/main/wiki/Finders#eager-loading) the meta when
querying for your models if you intend to use this method, to avoid the
[n+1 query problem](http://www.phabricator.com/docs/phabricator/article/Performance_N+1_Query_Problem.html)

The method accepts two parameters, $key and $raw. Passing in $key returns the meta value where meta\_key matches the
provided key. Passing in true for the $raw parameter will return an array of actual meta objects, as opposed to the hydrated
meta\_values.

###public function hydrate_meta

When working with ActiveRecord Models, WordPress will not serialize and unserialize data automatically for you. This method
accepts two parameters, meta_key and meta_value, and gives you the opportunity to modify meta values as necessary when
retreiving them using the $model->meta() method. (This does not apply when the $raw parameter is passed as true.)

Here's an example that will unserialize meta when the key is equal to 'serialized_meta':

    <?php
        
        public function hydrate_meta($meta_key, $meta_value)
        {
            return ($meta_key === 'serialized_meta') ?
                unserialize($meta_value) : $meta_value;
        }
        
###public function dehydrate_meta

This method serves the opposite purpose of hydrate_meta, and will be invoked on any values assigned to the 'meta_value' property
of a meta object. For a class with dehydrate_meta defined as follows:

    <?php
    
        public function dehydrate_meta($meta_key, $meta_value)
        {
            return ($meta_key === 'serialized_meta') ?
                serialize($meta_value) : $meta_value;
        }
        
Calling the below on a model instance of the class would make the following boolean statment true:

    <?php
    
        $model->meta_value = array('one', 'two', 'three');
        $model->meta_value === 'a:3:{i:0;s:3:"one";i:1;s:3:"two";i:2;s:5:"three";}';

##Views

***

swpMVC Views are based off of an older version of [Gabor DeMooij's Stamp library,](https://github.com/gabordemooij/stamp/blob/StampEngine/Stamp.php)
using extremely simple principles. Your views will contain no logic at all, and in most cases will be completely valid HTML on their own.

###Stamp tags

Stamp tags take the form of html comments, with an opening and closing comment representing one replaceable block. For example:

    <a href="<!-- url --><!-- /url -->">Link to somewhere</a>
    
gives you a region labeled url that can be manipulated from the stamp object.

###$stamp->replace()

If you place the above template code in a file called template.tpl, and call the below code from your controller:

    <?php
        
        echo $this->template('template')->replace('url', 'http://www.somesite.com');
        
the result would be:

    <a href="http://www.somesite.com">Link to somewhere</a>
    
###$stamp->copy()

This method allows you to copy defined template regions. Given the below template in file template.tpl:

    <p>Here's some stuff</p>
    <!-- more_stuff -->
        <p>And here's some more stuff</p>
    <!-- /more_stuff -->
    
The below code would yield true at the boolean in the last statement:

    <?php
    
        $more_stuff = $this->template('template')->copy('more_stuff');
        $more_stuff === "<p>And here's some more stuff</p>";


This is useful when populating one view with multiple models. For example, given the below template in file post.tpl:

    <h1><!-- post_title --><!-- /post_title --></h1>
    <!-- author_data -->
        by <!-- display_name --><!-- /display_name -->
    <!-- /author_data -->
    
The below controller code would replace post_title with the title of the post object, and display_name with the display name
of the post author:

    <?php
        
        $post = Post::first(array('include' => 'user');
        echo $post->render($this->template('post'))
            ->replace('author_data',
                $post->user->render(
                    $this->template('post')->copy('author_data')
                )
            );
    
###Populating views with $model->render()

When using the [$model->render()](/Streetwise-Wordpress-MVC/#models/model-render) method, your model will automatically replace tags named according
to the following conventions:

    <!-- attribute_name --><!-- /attribute_name -->

The above gets replaced with a model property named attribute\_name, or the return value of model instance method
    render\_attribute\_name if such method exists.
    
    <!-- attribute_name_block -->
        <!-- attribute_name --><!-- /attribute_name -->
    <!-- /attribute_name_block -->
    
The above gets replaced with a model property named attribute\_name, or the return value of model instance method
render\_attribute\_name if such method exists. If the value returns true when passed through $model->needs_template_cleanup(),
the entire attribute\_name\_block section is stripped from the template when rendered.
    
    <!-- control_attribute_name --><!-- /control_attribute_name -->

The above gets replaced with the 'attribute\_name' element of the array returned by static class method
[controls](/Streetwise-Wordpress-MVC/#models/public-static-function-controls)
    
    <!-- control_label_attribute_name --><!-- /control_label_attribute_name -->

The above gets replaced with the value of the 'label' key under the 'attribute\_name' key of the array returned by static class method controls

##Controllers

***

With well defined models, controllers can be relatively sparse. Your controller classes must extend swpMVCBaseController, which
equips them with the following functionality.

###$this->page_title

Set this property in your controller method to what you want the title attribute on the generated page to be. This needs to be set
before you call get\_header().

###$this->_templatedir

Set this property to the directory where your views for the current controller are stored, including a trailing slash.
Best practice is to define this in the constructor, in which case you'll want to make sure to call the parent constructor as well:

    <?php
        
        public function __construct()
        {
            $this->_templatedir = dirname(__FILE__).'/../views/';
            parent::__construct();
        }
        
###$this->_scripts

Set this property to an array where each element is an array of arguments to be passed to
[wp_enqueue_script](http://codex.wordpress.org/Function_Reference/wp_enqueue_script). Define the property before calling
get\_header() to have your scripts automatically enqueued on that page.

###$this->_styles

Same as $this->\_scripts, except each element of this array should be an array of arguments for
[wp_enqueue_style](http://codex.wordpress.org/Function_Reference/wp_enqueue_style).

###$this->_script_localizations

Same as \_styles and \_scripts, except each element of this array should be an array of arguments for
[wp_localize_script](http://codex.wordpress.org/Function_Reference/wp_localize_script).

###$this->template()

Requires $this->\_templatedir to be defined. Accepts the filename of a template (minus the file extension, which must be .tpl,)
and returns a [Stamp view](/Streetwise-Wordpress-MVC/#views) object for population and rendering. Here's an example of using the template method to
pass a view to a models render method:

    <?php
        
        $post = Post::first();
        $post->render($this->template('show_post'));
        
In the above example, we assume that the controllers templatedir property points to a directory that contains a file called
show_post.tpl, which contains the correct [Stamp tags](/Streetwise-Wordpress-MVC/#views/stamp-tags) to be populated by the Post model.

###$this->set404()

This method generates a WordPress 404 page using the currently selected WordPress theme. It must be called before any output
is generated. Here's an example of using this method within a controller method:

    <?php
    
        public function show_post($slug=false)
        {
            if (!$slug) return $this->set404();
            $post = Post::first(array('conditions' => array('post_name = ?', $slug)));
            if (!$post) return $this->set404();
            get_header();
            $post->render($this->template('show_post'));
            get_footer();
        }
        
In this method, if no slug was passed to the controller method, we return a 404. If a slug was passed, we attempt to find a post
using the provided slug. If we cannot, we return a 404. Only at that point if we have found a post using the provided slug do we
begin to generate output from the controller method.

###self::link()

This method accepts three arguments, a controller class name, a method name, and an optional array of parameters. It will then
return the corresponding url for that controller method. For example, if I've defined the following
[route](/Streetwise-Wordpress-MVC/#router/adding-routes) in my plugin:
    
    <?php
    
        $route = array('controller' => 'ControllerClass',
                            'method' => 'ControllerMethod',
                            'route' => '/url/of/route/:p/:p'
                        );
                        
Then the below statement would be true:

    <?php
    
        ControllerClass::link(
                    'ControllerClass',
                    'ControllerMethod', 
                    array('arg1', 'arg2')
                ) === get_bloginfo('url').'/url/of/route/arg1/arg2';
                
The link method is preferable to hard coding any fragment of a url into your views or controllers, since this will automatically update
any references if you change the route definitions for your plugin.

###$this->logError()

This method accepts a string as a parameter, and will write that string as an E\_USER\_WARNING level error to your PHP log,
as well as an error notice to the [pQp Console](/Streetwise-Wordpress-MVC/#logging-utility/php-quick-profiler) if you are running
in the development environment.

###public function before()

This method will run before any controller method is executed.

###public function after()

This method will run after any controller method is executed

###protected static $_cache

Set this variable on your controller class to stash queried data for accessing via other parts of your codebase. For example to
access a post queried from your controller method from a sidebar widget without setting a global variable or running a second
query, in your controller use:

    <?php
    
        $post = Post::first();
        self::$_cache['post'] = $post;

And then in your widget code you can use the following:

    <?php
    
        $cache = ControllerClass::cache();
        $post = $cache['post'];
        
Note you must use the static cache() method to retrieve the value. This prevents outside sources from polluting the data
cached by your controller.

##Logging/Utility

***

Several features have been incorporated into the framework to further improve the development experience.

###PHP Quick Profiler

The [PHP Quick Profiler](https://github.com/particletree/PHP-Quick-Profiler) is included with the framework. To activate,
add the following code to your wp-config file:

    <?php
        
        define('SW_WP_ENVIRONMENT', 'development');
        
The pQp will show on the bottom of every page, and by default will log load time, memory usage, file includes, and all SQL queries.

The following additional methods are available to write debug info to the console:

####Console::log()

Accepts two parameters. First is a variable to be dumped, second is an optional name.
This writes a general information entry to the console.

####Console::error()

Accepts same parameters as Console::log(). This writes an error entry to the console.

While best practice is to clean up any debug code you write after you don't need it anymore, leaving calls to Console::log() or
Console::error() in your code will not cause any issues if you run that code without your SW\_WP\_ENVIRONMENT set to development.

###SW_LOG_QUERIES

In cases where you want to debug queries that cannot be seen by the pQp (typically queries run on ajax calls,) the following code will
cause all queries to be written to your PHP log at level E\_USER\_WARNING:

    <?php
    
        define('SW_LOG_QUERIES', true);
        
You can then set this to false again to only log a specific set of queries within your code.


###Underscore PHP

The [Underscore PHP library](http://brianhaveri.github.com/Underscore.php/)
is an immensely useful set of tools for eliminating bulky boilerplate code used for common data
manipulations in favor of more elegant, concise alternatives, largely taking advantage of the closure support introduced in
PHP > 5.3. To avoid naming clashes with WordPress' \_\_ function, the included copy has been modified and can be accessed
only via the class methods, using a single underscore to reference the class. For example:

    <?php
    
        echo _::reduce(array('that\'s', 'all', 'for', 'now'),
            function($memo, $a) { return $memo.' '.$a; }, '');
        
##Credits

The following open source works have been instrumental in the creation of swpMVC, either as starting points, or having been
included in their entirety within the framework:

*   Stamp by [Gabor DeMooij](http://www.gabordemooij.com/)
*   [PHP Quick Profiler](https://github.com/particletree/PHP-Quick-Profiler), created by Ryan Campbell and designed by Kevin Hale
*   [WP-Developer-Tools](http://wordpress.org/extend/plugins/wp-developer-tools/) by [PHKCorp](http://profiles.wordpress.org/phkcorp2005/)
*   [PHP ActiveRecord](http://www.phpactiverecord.org/) by [Clay vanSchalkwijk, Jacques Fuentes, and Kien La](http://www.phpactiverecord.org/welcome/about)
*   [Underscore PHP](http://brianhaveri.github.com/Underscore.php/) by [Brian Haveri](https://github.com/brianhaveri)