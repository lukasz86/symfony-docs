.. index::
    single: Dependency Injection

The Dependency Injection Component
==================================

    The Dependency Injection component allows you to standardize and centralize
    the way objects are constructed in your application.

For an introduction to Dependency Injection and service containers see
:doc:`/book/service_container`

Installation
------------

You can install the component in many different ways:

* Use the official Git repository (https://github.com/symfony/DependencyInjection);
* Install it via PEAR ( `pear.symfony.com/DependencyInjection`);
* Install it via Composer (`symfony/dependency-injection` on Packagist).

Basic Usage
-----------

You might have a simple class like the following ``Mailer`` that
you want to make available as a service:

.. code-block:: php

    class Mailer
    {
        private $transport;

        public function __construct()
        {
            $this->transport = 'sendmail';
        }

        // ...
    }

You can register this in the container as a service:

.. code-block:: php

    use Symfony\Component\DependencyInjection\ContainerBuilder;

    $sc = new ContainerBuilder();
    $sc->register('mailer', 'Mailer');

An improvement to the class to make it more flexible would be to allow
the container to set the ``transport`` used. If you change the class
so this is passed into the constructor:

.. code-block:: php

    class Mailer
    {
        private $transport;

        public function __construct($transport)
        {
            $this->transport = $transport;
        }

        // ...
    }

Then you can set the choice of transport in the container:

.. code-block:: php

    use Symfony\Component\DependencyInjection\ContainerBuilder;

    $sc = new ContainerBuilder();
    $sc->register('mailer', 'Mailer')
        ->addArgument('sendmail'));

This class is now much more flexible as we have separated the choice of
transport out of the implementation and into the container.

Which mail transport you have chosen may be something other services need to
know about. You can avoid having to change it in multiple places by making
it a parameter in the container and then referring to this parameter for the
``Mailer`` service's constructor argument:


.. code-block:: php

    use Symfony\Component\DependencyInjection\ContainerBuilder;

    $sc = new ContainerBuilder();
    $sc->setParameter('mailer.transport', 'sendmail');
    $sc->register('mailer', 'Mailer')
        ->addArgument('%mailer.transport%'));

Now that the ``mailer`` service is in the container you can inject it as
a dependency of other classes. If you have a ``NewsletterManager`` class
like this:

.. code-block:: php

    use Mailer;

    class NewsletterManager
    {
        private $mailer;

        public function __construct(Mailer $mailer)
        {
            $this->mailer = $mailer;
        }

        // ...
    }

Then you can register this as a service as well and pass the ``mailer`` service into it:

.. code-block:: php

    use Symfony\Component\DependencyInjection\ContainerBuilder;
    use Symfony\Component\DependencyInjection\Reference;

    $sc = new ContainerBuilder();

    $sc->setParameter('mailer.transport', 'sendmail');
    $sc->register('mailer', 'Mailer')
        ->addArgument('%mailer.transport%'));

    $sc->register('newsletter_manager', 'NewsletterManager')
        ->addArgument(new Reference('mailer'));

If the ``NewsletterManager`` did not require the ``Mailer`` and injecting
it was only optional then you could use setter injection instead:

.. code-block:: php

    use Mailer;

    class NewsletterManager
    {
        private $mailer;

        public function setMailer(Mailer $mailer)
        {
            $this->mailer = $mailer;
        }

        // ...
    }

You can now choose not to inject a ``Mailer`` into the ``NewsletterManager``.
If you do want to though then the container can call the setter method:

.. code-block:: php

    use Symfony\Component\DependencyInjection\ContainerBuilder;
    use Symfony\Component\DependencyInjection\Reference;

    $sc = new ContainerBuilder();

    $sc->setParameter('mailer.transport', 'sendmail');
    $sc->register('mailer', 'Mailer')
        ->addArgument('%mailer.transport%'));

    $sc->register('newsletter_manager', 'NewsletterManager')
        ->addMethodCall('setMailer', new Reference('mailer'));

You could then get your ``newsletter_manager`` service from the container
like this:

.. code-block:: php

    use Symfony\Component\DependencyInjection\ContainerBuilder;
    use Symfony\Component\DependencyInjection\Reference;

    $sc = new ContainerBuilder();

    //--

    $newsletterManager = $sc->get('newsletter_manager');

Avoiding Your Code Becoming Dependent on the Container
------------------------------------------------------

Whilst you can retrieve services from the container directly it is best
to minimize this. For example, in the ``NewsletterManager`` we injected
the ``mailer`` service in rather than asking for it from the container.
We could have injected the container in and retrieved the ``mailer`` service
from it but it would then be tied to this particular container making it
difficult to reuse the class elsewhere.

You will need to get a service from the container at some point but this
should be as few times as possible at the entry point to your application.

Setting Up the Container with Configuration Files
-------------------------------------------------

As well as setting up the services using PHP as above you can also use configuration
files. To do this you also need to install the Config component:

* Use the official Git repository (https://github.com/symfony/Config);
* Install it via PEAR ( `pear.symfony.com/Config`);
* Install it via Composer (`symfony/config` on Packagist).

Loading an xml config file:

.. code-block:: php

    use Symfony\Component\DependencyInjection\ContainerBuilder;
    use Symfony\Component\Config\FileLocator;
    use Symfony\Component\DependencyInjection\Loader\XmlFileLoader;

    $sc = new ContainerBuilder();
    $loader = new XmlFileLoader($container, new FileLocator(__DIR__));
    $loader->load('services.xml');

Loading a yaml config file:

.. code-block:: php

    use Symfony\Component\DependencyInjection\ContainerBuilder;
    use Symfony\Component\Config\FileLocator;
    use Symfony\Component\DependencyInjection\Loader\YamlFileLoader;

    $sc = new ContainerBuilder();
    $loader = new YamlFileLoader($container, new FileLocator(__DIR__));
    $loader->load('services.yml');

The ``newsletter_manager`` and ``mailer`` services can be set up using config files:

.. configuration-block::

    .. code-block:: yaml

        # src/Acme/HelloBundle/Resources/config/services.yml
        parameters:
            # ...
            mailer.transport: sendmail

        services:
            my_mailer:
                class:     Mailer
                arguments: [@mailer]
            newsletter_manager:
                class:     NewsletterManager
                calls:
                    - [ setMailer, [ @mailer ] ]

    .. code-block:: xml

        <!-- src/Acme/HelloBundle/Resources/config/services.xml -->
        <parameters>
            <!-- ... -->
            <parameter key="mailer.transport">sendmail</parameter>
        </parameters>

        <services>
            <service id="mailer" class="Mailer">
                <argument>%mailer.transport%</argument>
            </service>

            <service id="newsletter_manager" class="NewsletterManager">
                <call method="setMailer">
                     <argument type="service" id="mailer" />
                </call>
            </service>
        </services>

    .. code-block:: php

        use Symfony\Component\DependencyInjection\Reference;

        // ...
        $sc->setParameter('mailer.transport', 'sendmail');
        $sc->register('mailer', 'Mailer')
           ->addArgument('%mailer.transport%'));

        $sc->register('newsletter_manager', 'NewsletterManager')
           ->addMethodCall('setMailer', new Reference('mailer'));


Learn more from the Cookbook
----------------------------

* :doc:`/cookbook/service_container/factories`
* :doc:`/cookbook/service_container/parentservices`
