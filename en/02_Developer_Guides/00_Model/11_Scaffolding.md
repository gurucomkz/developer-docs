---
title: Building Model and Search Interfaces around Scaffolding
summary: A Model-driven approach to defining your application UI.
icon: hammer
---

# Scaffolding

The ORM already has a lot of information about the data represented by a `DataObject` through its `$db` property, so 
Silverstripe CMS will use that information to scaffold some interfaces. This is done though [FormScaffolder](api:SilverStripe\Forms\FormScaffolder)
to provide reasonable defaults based on the property type (e.g. a checkbox field for booleans). You can then further 
customise those fields as required.

## Form Fields

An example is `DataObject`, Silverstripe CMS will automatically create your CMS interface so you can modify what you need.

```php
use SilverStripe\ORM\DataObject;

class MyDataObject extends DataObject 
{
    
    private static $db = [
        'IsActive' => 'Boolean',
        'Title' => 'Varchar',
        'Content' => 'Text'
    ];

    public function getCMSFields() 
    {
        // parent::getCMSFields() does all the hard work and creates the fields for Title, IsActive and Content.
        $fields = parent::getCMSFields();
        $fields->dataFieldByName('IsActive')->setTitle('Is active?');
        
        return $fields;
    }
}
```

To fully customise your form fields, start with an empty FieldList.

```php
public function getCMSFields() 
{
    $fields = FieldList::create(
        TabSet::create("Root",
            Tab::create("Main",
                CheckboxSetField::create('IsActive','Is active?'),
                TextField::create('Title'),
                TextareaField::create('Content')
                    ->setRows(5)
            )
        )
    );
    
    return $fields;
}
```

You can also alter the fields of built-in and module `DataObject` classes through your own 
[DataExtension](/developer_guides/extending/extensions), and a call to `DataExtension->updateCMSFields`.

[info]
`FormField` scaffolding takes [`$field_labels` config](#field-labels) into account as well.
[/info]

## Searchable Fields

The `$searchable_fields` property uses a mixed array format that can be used to further customise your generated admin
system. The default is a set of array values listing the fields.

[info]
`$searchable_fields` will default to use the [`$summary_fields` config](#summary-fields) if not defined. This works fine unless 
your `$summary_fields` config specifies fields that are not stored in the database.
[/info]

```php
use SilverStripe\ORM\DataObject;

class MyDataObject extends DataObject 
{

   private static $searchable_fields = [
      'Name',
      'ProductCode'
   ];
}
```

### Specify a form field or search filter

Searchable fields will appear in the search interface with a default form field (usually a [TextField](api:SilverStripe\Forms\TextField)) and a 
default search filter assigned (usually an [ExactMatchFilter](api:SilverStripe\ORM\Filters\ExactMatchFilter)). To override these defaults, you can specify 
additional information on `$searchable_fields`:

```php
use SilverStripe\ORM\DataObject;

class MyDataObject extends DataObject 
{

    private static $searchable_fields = [
        'Name' => 'PartialMatchFilter',
        'ProductCode' => NumericField::class
    ];
}
```

If you assign a single string value, you can set it to be either a [FormField](api:SilverStripe\Forms\FormField) or [SearchFilter](api:SilverStripe\ORM\Filters\SearchFilter). To specify 
both, you can assign an array:

```php
use SilverStripe\ORM\DataObject;

class MyDataObject extends DataObject 
{

   private static $searchable_fields = [
       'Name' => [
          'field' => TextField::class,
          'filter' => 'PartialMatchFilter',
       ],
       'ProductCode' => [
           'title' => 'Product code #',
           'field' => NumericField::class,
           'filter' => 'PartialMatchFilter',
       ],
   ];
}
```

### Searching on relations

To include relations (`$has_one`, `$has_many` and `$many_many`) in your search, you can use a dot-notation.

```php
use SilverStripe\ORM\DataObject;

class Team extends DataObject 
{
    private static $db = [
        'Title' => 'Varchar'
    ];
    
    private static $many_many = [
        'Players' => 'Player'
    ];
    
    private static $searchable_fields = [
        'Title',
        'Players.Name',
    ];
}

class Player extends DataObject 
{
    private static $db = [
        'Name' => 'Varchar',
        'Birthday' => 'Date',
    ];
    
    private static $belongs_many_many = [
        'Teams' => 'Team'
    ];
}

```

### Searching many db fields on a single search field

Use a single search field that matches on multiple database fields with `'match_any'`. This also supports specifying a field and a filter, though it is not necessary to do so.

```php
class Order extends DataObject
{
    private static $db = [
        'Name' => 'Varchar',
    ];

    private static $has_one = [
        'Customer' => Customer::class,
        'ShippingAddress' => Address::class,
    ];

    private static $searchable_fields = [
        'CustomName' => [
            'title' => 'First Name',
            'field' => TextField::class,
            'match_any' => [
                // Searching with the "First Name" field will show Orders matching either Name, Customer.FirstName, or ShippingAddress.FirstName
                'Name',
                'Customer.FirstName',
                'ShippingAddress.FirstName',
            ]
        ]
    ];
}
```

[alert]
If you don't specify a field, you must use the name of a real database field instead of a custom name so that a default field can be determined.
[/alert]

## Summary Fields

Summary fields can be used to show a quick overview of the data for a specific [DataObject](api:SilverStripe\ORM\DataObject) record. The most common use 
is their display as table columns, e.g. in the search results of a [ModelAdmin](api:SilverStripe\Admin\ModelAdmin) CMS interface.

```php
use SilverStripe\ORM\DataObject;

class MyDataObject extends DataObject 
{
    private static $db = [
        'Name' => 'Text',
        'OtherProperty' => 'Text',
        'ProductCode' => 'Int',
    ]; 
    
    private static $summary_fields = [
        'Name',
        'ProductCode',
    ];
}
```

### Relations in summary fields

To include relations or field manipulations in your summaries, you can use a dot-notation.

```php
use SilverStripe\ORM\DataObject;

class OtherObject extends DataObject 
{    
    private static $db = [
        'Title' => 'Varchar',
    ];
}

class MyDataObject extends DataObject 
{
    private static $db = [
        'Name' => 'Text',
        'Description' => 'HTMLText',
    ];
    
    private static $has_one = [
        'OtherObject' => 'OtherObject',
    ];
    
    private static $summary_fields = [
        'Name' => 'Name',
        'Description.Summary' => 'Description (summary)',
        'OtherObject.Title' => 'Other Object Title',
    ];
}

```

### Images in summary fields

Non-textual elements (such as images and their manipulations) can also be used in summaries.

```php
use SilverStripe\ORM\DataObject;

class MyDataObject extends DataObject 
{   
    private static $db = [
        'Name' => 'Text',
    ];
    
    private static $has_one = [
        'HeroImage' => 'Image',
    ];
    
    private static $summary_fields = [
        'Name' => 'Name',
        'HeroImage.CMSThumbnail' => 'Hero Image',
    ];
}

```

## Field labels

In order to re-label any summary fields, you can use the `$field_labels` static. This will also affect the output of `$object->fieldLabels()` and `$object->fieldLabel()`.

```php
use SilverStripe\ORM\DataObject;

class MyDataObject extends DataObject 
{   
    private static $db = [
        'Name' => 'Text',
    ];
    
    private static $has_one = [
        'HeroImage' => 'Image',
    ];
    
    private static $summary_fields = [
        'Name',
        'HeroImage.CMSThumbnail',
    ];
    
    private static $field_labels = [
        'Name' => 'Name',
        'HeroImage.CMSThumbnail' => 'Hero',
    ];
}
```

## Related Documentation

* [SearchFilters](searchfilters)

## API Documentation

* [FormScaffolder](api:SilverStripe\Forms\FormScaffolder)
* [DataObject](api:SilverStripe\ORM\DataObject)
