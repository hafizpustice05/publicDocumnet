Api documentation generate package [Git hub link](https://github.com/ovac/idoc?ref=madewithlaravel.com), briefly described in that link.  


## Override this package
iDocs package cann't support multiple response add. so we want to add multiple response.

Create `App\MyCustom\IDoc\IDocGeneratorCommand` `Class` this directory & paste this code there.

```php
namespace App\MyCustom\IDoc;

use OVAC\IDoc\IDocGenerator;
use Illuminate\Console\Command;
use Illuminate\Routing\Route;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\File;
use Mpociot\Reflection\DocBlock;
use OVAC\IDoc\Tools\RouteMatcher;
use ReflectionClass;
use ReflectionException;


class IDocGeneratorCommand extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'idoc:generated
                            {--force : Force rewriting of existing routes}
    ';



    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Generate interractive api documentation.';

    private $routeMatcher;

    public function __construct(RouteMatcher $routeMatcher)
    {
        parent::__construct();
        $this->routeMatcher = $routeMatcher;
    }

    /**
     * Execute the console command.
     *
     * @return void
     */
    public function handle()
    {
        $usingDingoRouter = strtolower(config('idoc.router')) == 'dingo';
        if ($usingDingoRouter) {
            $routes = $this->routeMatcher->getDingoRoutesToBeDocumented(config('idoc.routes'));
        } else {
            $routes = $this->routeMatcher->getLaravelRoutesToBeDocumented(config('idoc.routes'));
        }

        $generator = new IDocGenerator();

        $parsedRoutes = $this->processRoutes($generator, $routes);

        $parsedRoutes = collect($parsedRoutes)->groupBy('group');

        $this->writeMarkdown($parsedRoutes);
    }

    /**
     * @param  Collection $parsedRoutes
     *
     * @return void
     */
    private function writeMarkdown($parsedRoutes)
    {
        $outputPath = public_path(config('idoc.output'));

        if (!File::isDirectory($outputPath)) {
            File::makeDirectory($outputPath, 0777, true, true);
        }

        $this->info('Generating OPEN API 3.0.0 Config');
        file_put_contents($outputPath . DIRECTORY_SEPARATOR . 'openapi.json', $this->generateOpenApi3Config($parsedRoutes));
    }

    /**
     * @param IDocGenerator $generator
     * @param array $routes
     *
     * @return array
     */
    private function processRoutes(IDocGenerator $generator, array $routes)
    {
        $parsedRoutes = [];
        foreach ($routes as $routeItem) {
            $route = $routeItem['route'];
            /** @var Route $route */
            if ($this->isValidRoute($route) && $this->isRouteVisibleForDocumentation($route->getAction()['uses'])) {
                $parsedRoutes[] = $generator->processRoute($route, $routeItem['apply']);
                $this->info('Processed route: [' . implode(',', $generator->getMethods($route)) . '] ' . $generator->getUri($route));
            } else {
                $this->warn('Skipping route: [' . implode(',', $generator->getMethods($route)) . '] ' . $generator->getUri($route));
            }
        }

        return $parsedRoutes;
    }

    /**
     * @param $route
     *
     * @return bool
     */
    private function isValidRoute(Route $route)
    {
        return !is_callable($route->getAction()['uses']) && !is_null($route->getAction()['uses']);
    }

    /**
     * @param $route
     *
     * @throws ReflectionException
     *
     * @return bool
     */
    private function isRouteVisibleForDocumentation($route)
    {
        list($class, $method) = explode('@', $route);
        $reflection = new ReflectionClass($class);

        if (!$reflection->hasMethod($method)) {
            return false;
        }

        $comment = $reflection->getMethod($method)->getDocComment();

        if ($comment) {
            $phpdoc = new DocBlock($comment);

            return collect($phpdoc->getTags())
                ->filter(function ($tag) use ($route) {
                    return $tag->getName() === 'hideFromAPIDocumentation';
                })
                ->isEmpty();
        }

        return true;
    }


    /**
     * Generate Open API 3.0.0 collection json file.
     *
     * @param Collection $routes
     *
     * @return string
     */
    public function generateOpenApi3Config(Collection $routes)
    {
        $result = $routes->map(function ($routeGroup, $groupName) use ($routes) {

            return collect($routeGroup)->map(function ($route) use ($groupName, $routes, $routeGroup) {

                $methodGroup = $routeGroup->where('uri', $route['uri'])->mapWithKeys(function ($route) use ($groupName, $routes) {

                    $bodyParameters = collect($route['bodyParameters'])->map(function ($schema, $name) use ($routes) {

                        $type = $schema['type'];
                        $default = $schema['value'];

                        if ($type === 'float') {
                            $type = 'number';
                        }

                        if ($type === 'json' && $default) {
                            $type = 'object';
                            $default = json_decode($default);
                        }

                        return [
                            'in' => 'formData',
                            'name' => $name,
                            'description' => $schema['description'],
                            'required' => $schema['required'],
                            'type' => $type,
                            'default' => $default,
                        ];
                    });

                    $jsonParameters = [
                        'application/json' => [
                            'schema' => [
                                'type' => 'object',
                            ]
                                + (count($required = $bodyParameters
                                    ->values()
                                    ->where('required', true)
                                    ->pluck('name'))
                                    ? ['required' => $required]
                                    : [])

                                + (count($properties = $bodyParameters
                                    ->values()
                                    ->filter()
                                    ->mapWithKeys(function ($parameter) use ($routes) {
                                        return [
                                            $parameter['name'] => [
                                                'type' => $parameter['type'],
                                                'example' => $parameter['default'],
                                                'description' => $parameter['description'],
                                            ],
                                        ];
                                    }))
                                    ? ['properties' => $properties]
                                    : [])

                                + (count($properties = $bodyParameters
                                    ->values()
                                    ->filter()
                                    ->mapWithKeys(
                                        function ($parameter) {
                                            return [$parameter['name'] => $parameter['default']];
                                        }
                                    ))
                                    ? ['example' => $properties]
                                    : [])
                        ],
                    ];

                    $queryParameters = collect($route['queryParameters'])->map(function ($schema, $name) {
                        return [
                            'in' => 'query',
                            'name' => $name,
                            'description' => $schema['description'],
                            'required' => $schema['required'],
                            'schema' => [
                                'type' => $schema['type'],
                                'example' => $schema['value'],
                            ],
                        ];
                    });

                    $pathParameters = collect($route['pathParameters'] ?? [])->map(function ($schema, $name) use ($route) {
                        return [
                            'in' => 'path',
                            'name' => $name,
                            'description' => $schema['description'],
                            'required' => $schema['required'],
                            'schema' => [
                                'type' => $schema['type'],
                                'example' => $schema['value'],
                            ],
                        ];
                    });

                    $headerParameters = collect($route['headers'])->map(function ($value, $header) use ($route) {

                        if ($header === 'Authorization') {
                            return;
                        }

                        return [
                            'in' => 'header',
                            'name' => $header,
                            'description' => '',
                            'required' => true,
                            'schema' => [
                                'type' => 'string',
                                'default' => $value,
                                'example' => $value,
                            ],
                        ];
                    });

                    return [
                        strtolower($route['methods'][0]) => (

                            ($route['authenticated']
                                ? ['security' => [
                                    collect(config('idoc.security'))->map(function () {
                                        return [];
                                    }),
                                ]]
                                : [])

                            + ([
                                "tags" => [
                                    $groupName,
                                ],
                                'operationId' => $route['title'],
                                'description' => $route['description'],
                            ]) +

                            (count(array_intersect(['POST', 'PUT', 'PATCH'], $route['methods']))
                                ? ['requestBody' => [
                                    'description' => $route['description'],
                                    'required' => true,
                                    'content' => collect($jsonParameters)->filter()->toArray(),
                                ]]
                                : []) +

                            [
                                'parameters' => (array_merge(
                                    collect($queryParameters->values()->toArray())
                                        ->filter()
                                        ->toArray(),
                                    collect($pathParameters->values()->toArray())
                                        ->filter()
                                        ->toArray()
                                ) +

                                    collect($headerParameters->values()->toArray())
                                    ->filter()
                                    ->values()
                                    ->toArray()),

                                'responses' => $this->multiResponse($route['response']),

                                'x-code-samples' => collect(config('idoc.language-tabs'))->map(function ($name, $lang) use ($route) {
                                    return [
                                        'lang' => $name,
                                        'source' => view('idoc::languages.' . $lang, compact('route'))->render(),
                                    ];
                                })->values()->toArray(),
                            ]),
                    ];
                });

                return collect([
                    ('/' . $route['uri']) => $methodGroup,
                ]);
            });
        });

        $paths = [];

        foreach ($result->filter()->toArray() as $groupName => $group) {
            foreach ($group as $key => $value) {
                $paths[key($value)] = $value[key($value)];
            }
        }

        $collection = [

            'openapi' => '3.0.0',

            'info' => [
                'title' => config('idoc.title'),
                'version' => config('idoc.version'),
                'description' => config('idoc.description'),
                'termsOfService' => config('idoc.terms_of_service'),
                "license" =>  !empty(config('idoc.license')) ? config('idoc.license') : null,
                "contact" =>  config('idoc.contact'),
                "x-logo" => [
                    "url" => config('idoc.logo'),
                    "altText" => config('idoc.title'),
                    "backgroundColor" => config('idoc.color'),
                ],
            ],

            'components' => [

                'securitySchemes' => config('idoc.security'),

                'schemas' => $routes->mapWithKeys(function ($routeGroup, $groupName) {

                    if ($groupName != 'Payment processors') {
                        return [];
                    }

                    return collect($routeGroup)->mapWithKeys(function ($route) use ($groupName, $routeGroup) {

                        $bodyParameters = collect($route['bodyParameters'])->map(function ($schema, $name) {

                            $type = $schema['type'];

                            if ($type === 'float') {
                                $type = 'number';
                            }

                            if ($type === 'json') {
                                $type = 'object';
                            }

                            return [
                                'in' => 'formData',
                                'name' => $name,
                                'description' => $schema['description'],
                                'required' => $schema['required'],
                                'type' => $type,
                                'default' => $schema['value'],
                            ];
                        });

                        return [
                            "PM{$route['paymentMethod']->id}" => ['type' => 'object']

                                + (count($required = $bodyParameters
                                    ->values()
                                    ->where('required', true)
                                    ->pluck('name'))
                                    ? ['required' => $required]
                                    : [])

                                + (count($properties = $bodyParameters
                                    ->values()
                                    ->filter()
                                    ->mapWithKeys(function ($parameter) {
                                        return [
                                            $parameter['name'] => [
                                                'type' => $parameter['type'],
                                                'example' => $parameter['default'],
                                                'description' => $parameter['description'],
                                            ],
                                        ];
                                    }))
                                    ? ['properties' => $properties]
                                    : [])

                                + (count($properties = $bodyParameters
                                    ->values()
                                    ->filter()
                                    ->mapWithKeys(function ($parameter) {
                                        return [$parameter['name'] => $parameter['default']];
                                    }))
                                    ? ['example' => $properties]
                                    : [])
                        ];
                    });
                })->filter(),
            ],

            'servers' => config('idoc.servers'),

            'paths' => $paths,

            'x-tagGroups' => config('idoc.tag_groups'),
        ];

        return json_encode($collection);
    }

    public function multiResponse($route)
    {

        if (count($route) == 1) {

            return [$route[0]['status'] ?? 200 => [
                'description' => 'success',
            ] +
                (count($route ?? [])
                    ?
                    ['content' => [
                        'application/json' => [
                            'schema' => [
                                'type' => 'object',
                                'example' => json_decode($route[0]['content'], true),
                            ],
                        ],
                    ]]
                    : [])];
        } else if (count($route) == 2) {
            return [
                $route[0]['status'] ?? 200 => [
                    'description' => 'success',
                ] +
                    (count($route ?? [])
                        ?
                        ['content' => [
                            'application/json' => [
                                'schema' => [
                                    'type' => 'object',
                                    'example' => json_decode($route[0]['content'], true),
                                ],
                            ],
                        ]]
                        : []),
                $route[1]['status'] ?? 200 => [
                    'description' => 'success',
                ] +
                    (count($route ?? [])
                        ?
                        ['content' => [
                            'application/json' => [
                                'schema' => [
                                    'type' => 'object',
                                    'example' => json_decode($route[1]['content'], true),
                                ],
                            ],
                        ]]
                        : []),

            ];
        } else if (count($route) == 3) {
            return [
                $route[0]['status'] ?? 200 => [
                    'description' => 'success',
                ] +
                    (count($route ?? [])
                        ?
                        ['content' => [
                            'application/json' => [
                                'schema' => [
                                    'type' => 'object',
                                    'example' => json_decode($route[0]['content'], true),
                                ],
                            ],
                        ]]
                        : []),
                $route[1]['status'] ?? 200 => [
                    'description' => 'success',
                ] +
                    (count($route ?? [])
                        ?
                        ['content' => [
                            'application/json' => [
                                'schema' => [
                                    'type' => 'object',
                                    'example' => json_decode($route[1]['content'], true),
                                ],
                            ],
                        ]]
                        : []),

                $route[2]['status'] ?? 200 => [
                    'description' => 'success',
                ] +
                    (count($route ?? [])
                        ?
                        ['content' => [
                            'application/json' => [
                                'schema' => [
                                    'type' => 'object',
                                    'example' => json_decode($route[2]['content'], true),
                                ],
                            ],
                        ]]
                        : []),

            ];
        }
    }
    // json_decode($route[0]['content'], true)
}

```

Register this code app Service provider or any custom provider copy this code 

```php
 /**IDoc generator class override and register it */
$this->commands([
    \App\MyCustom\IDoc\IDocGeneratorCommand::class,
]);
 /** end of register idoc class */
```

Finally run this command
`php artisan idoc:generated`

> This code suport 3 response added successfully. You want to add more response, customize `multiResponse` function

