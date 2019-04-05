# Proposal for a PSR for message translator

The main aim of this proposal is to provide a common interface for the interoperability of message translations across different packages

## What is the scope?

* Provide a way to consume message translations in different languages.
* Allow handling plurals
* Allow using translations from different contexts and domains

## What is out of the scope?

* Numbers, currency, time, and other locale formats are not covered
* Message manipulation: Create/edit/delete translations. This proposal is focused only in consume translations. However other PSR might be created in the future to standardize the translation object itself.
* Placeholders in the translations. This proposal does not dictate a way to define placeholders (ex: `Hello %s` or `Hello :name`, or `Hello {name}`)

## Current libraries

This spec has taken the following libraries into account:

* [Gettext](https://github.com/oscarotero/Gettext/blob/master/src/TranslatorInterface.php)
* [Symfony](https://github.com/symfony/translation/blob/master/TranslatorInterface.php)
* [Zend](https://github.com/zendframework/zend-i18n/blob/master/src/Translator/TranslatorInterface.php)
* [Yii2](https://github.com/yiisoft/yii2/blob/master/framework/i18n/I18N.php)
* [Drupal](https://github.com/drupal/drupal/blob/563ef195530621f3f15a3a12b488f2ee230d8204/core/lib/Drupal/Core/StringTranslation/Translator/TranslatorInterface.php)
* [Concrete5](https://github.com/concrete5/concrete5-core/blob/develop/src/Localization/Translator/TranslatorAdapterInterface.php)
* [PhpBB](https://github.com/phpbb/phpbb-core/blob/master/language/language.php)

## TranslatorInterface

```php
namespace Psr\I18n\Messages

interface TranslatorInterface
{
    /**
     * Get the translation of a message
     * 
     * @param  string      $message The text to translate
     * @param  string|null $context The context if exists
     * 
     * @return string|null
     */
    public function translate($message, $context = null);

    /**
     * Get the translation of a message in the correct plural form
     * 
     * @param  integer     $count   The counter used to evaluate the plural
     * @param  string      $message The text to translate (in singular)
     * @param  string|null $context The context if exists
     * 
     * @return string|null
     */
    public function translatePlural($count, $message, $context = null);
}
```

## Simple translations

The `TranslatorInterface` is an interface with a simple API to get translations using the original text.

```php
$translator->translate('Hello world'); // Ola mundo
$translator->translate('not-found-translation'); // NULL
```

As you can see, if the translation does not exist, returns `null`. This is a convenient way to know if a translation is missing or exist but is empty.

Sometimes, the same text is used in different places, with different contexts and can produce different translations. To solve these ambiguities, there's a second optional argument to include a context.

```php
$translator->translate('Save', 'new-comment'); //Crear comentario
$translator->translate('Save', 'edit-comment'); //Editar comentario
```

## Plurals

If a text contains numbers or counters, it may have different versions: a singular form and one or various plurals (depending on the language). To cover this case, there's the `translatePlural` method:

```php
$translator->translatePlural(1, 'Save item'); //Gardar item
$translator->translatePlural(5, 'Save item'); //Gardar items
```

Like simple translations, there's an extra optional argument for contexts:

```php
$translator->translatePlural(1, 'Save item', 'new-comment'); //Crear comentario
$translator->translatePlural(2, 'Save item', 'new-comment'); //Crear comentarios

$translator->translatePlural(1, 'Save item', 'edit-comment'); //Editar comentario
$translator->translatePlural(2, 'Save item', 'edit-comment'); //Editar comentarios
```

## Domains

To handle translations from other domains, you can use other instances of `TranslatorInterface`. For simplicity, this interface cannot handle different domains in the same instance.

## Usage

This proposal defines an API intended to be used by any library. However, final consumers may have other different API providing less verbose and more functionalities (like placeholders support). Besides that, many template engines like twig, smarty, mustache, have their own syntax to handle multilanguage texts but can use a `TranslatorInterface` to access to the translated data.

Let's see an example of how to emulate [PHP gettext functions](http://php.net/manual/en/ref.gettext.php)

```php
class GettextEmulator
{
    private static $defaultDomain = 'app';
    private static $domains = [];

    private static function getTranslator($domain)
    {
        if (!isset(self::$domains[$domain])) {
            self::$domains[$domain] = new Translator($domain);
        }

        return self::$domains[$domain];
    }

    public static function textdomain($text_domain = null)
    {
        if ($text_domain === null) {
            return self::$defaultDomain;
        }

        self::$defaultDomain = $text_domain;
    }

    public static function gettext($message)
    {
        return self::dgettext(self::$defaultDomain, $message);
    }

    public static function ngettext($message, $plural, $number)
    {
        return self::ndgettext(self::$defaultDomain, $message, $plural, $number);
    }

    public static function dgettext($domain, $message)
    {
        $translation = self::getTranslator($domain)->translate($message);

        return is_null($translation) ? $message : $translation;
    }

    public static function dngettext($domain, $message, $plural, $number)
    {
        $translation = self::getTranslator($domain)->translatePlural($number, $message);

        return is_null($translation) ? ($number === 1 ? $message : $plural) : $translation;
    }
}
```

## FAQ

### Why the plural message is missing in `translatePlural`?

It's common to see the `translatePlural` accepting the singular, plural and the counter. For example:
```php
ngettext('One comment', 'Many comments', $count)
```
The plural is used only as fallback, but it does not affect to the uniqueness of the message in gettext. In other words: two translations with the same singular but different plurals (or one of them without plural) [**are the same translation.**](https://github.com/oscarotero/Gettext/issues/48) This makes this argument useless in the purpose of this standard.

Another reason is not all translator implementations are based in gettext. For example, [Symfony's plural method](https://github.com/symfony/translation/blob/master/TranslatorInterface.php#L50) does not have the plural argument. And there's some implementations that don't even have a way to handle plural translations (like Yii or Drupal).

### Why return `null` instead of a string with the original message if the translation does not exist?

Because the plural message (used as fallback) has been removed from `translatePlural`, so it cannot return the fallback if the message is plural.
