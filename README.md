# 从0开始写laravel-日志与异常处理

## 日志需求设计

1. 日志分3中类型:  info, debug, error
2. 日志信息根据时间日期不同，分别保存在不同的文件中

## 实现

在 system 中创建文件log.php， 别拥有四个方法


	<?php

	namespace System;

	class Log {

	    public static function info($message)
	    {
	        static::write('Info', $message);
	    }

	    public static function debug($message)
	    {
	        static::write('Debug', $message);
	    }

	    public static function error($message)
	    {
	        static::write('Error', $message);
	    }

	    public static function write($type, $message)
	    {

	    }
	}


程序需要根据不同的日期创建文件并以不同的文件夹进行归类，所有需要有创建文件的方法，增加Log类增加私有方法：

	private static function make_directory($directory)
    {
        mkdir($directory, 02777);

        chmod($directory, 02777);
    }

在 Log 类 write 方法中，先判断文件目录是否存在，否则创建，最后写文件，写文件的过程使用文件锁

	public static function write($type, $message)
    {
        $directory = APP_PATH.'logs/'.date('Y');

        if ( ! is_dir($directory))
        {
            static::make_directory($directory);
        }

        $directory .= '/'.date('m');

        if ( ! is_dir($directory))
        {
            static::make_directory($directory);
        }

        $file = $directory.'/'.date('d').EXT;

        // 使用文件锁，在文件最后新增字符内容
        file_put_contents($file, date('Y-m-d H:i:s').' '.$type.' - '.$message.PHP_EOL, LOCK_EX | FILE_APPEND);

        chmod($file, 0666);
    }



## 异常处理需求设计

1. 当系统遭遇异常或者错误的时候，由统一的地方进行处理
2. 页面显示错误信息与错误位置
3. 通过配置文件来控制是否将错误信息记录到日志和输出错误信息

## 实现

错误信息页面：

在 system 下创建 views 目录，在 system/views 下创建 error 目录， 在system/views/error 下创建视图文件：

	<!DOCTYPE html>
	<html>
	<head>
	    <meta charset="utf-8">
	    <title>Laravel - Error</title>

	    <link href='http://fonts.googleapis.com/css?family=Ubuntu&amp;subset=latin' rel='stylesheet' type='text/css'>

	    <style type="text/css">
	        body {
	            background-color: #fff;
	            font-family: 'Ubuntu', sans-serif;
	            font-size: 18px;
	            color: #3f3f3f;
	            padding: 10px;
	        }

	        h1 {
	            font-family: 'Ubuntu', sans-serif;
	            font-size: 45px;
	            color: #6d6d6d;
	            margin: 0 0 10px 0;
	        }

	        h3 {
	            color: #6d6d6d;
	            margin: 0 0 10px 0;
	        }

	        pre {
	            font-size: 14px;
	            margin: 0 0 0 0;
	            padding: 0 0 0 0;
	        }

	        #wrapper {
	            width: 100%;
	        }

	        div.content {
	            padding: 10px 10px 10px 10px;
	            background-color: #eee;
	            border-radius: 10px;
	            margin-bottom: 10px;
	        }
	    </style>
	</head>
	<body>
	<div id="wrapper">
	    <h1><?php echo $severity; ?></h1>

	    <div class="content">
	        <h3>Message:</h3>
	        <?php echo $message; ?> in <strong><?php echo basename($file); ?></strong> on line <strong><?php echo $line; ?></strong>.
	    </div>

	    <div class="content">
	        <h3>Stack Trace:</h3>

	        <pre><?php echo $trace; ?></pre>
	    </div>

	    <div class="content">
	        <h3>Context:</h3>

	        <?php if (count($contexts) > 0) { ?>

	            <?php foreach ($contexts as $num => $context) { ?>
	                <pre><?php echo htmlentities($num.' '.$context); ?></pre>
	            <?php } ?>

	        <?php } else { ?>
	            Context unavailable.
	        <?php } ?>
	    </div>
	</div>
	</body>
	</html>


在 public/index.php 下通过set_exception_handler(), set_error_handler(), register_shutdown_function()方法来注册异常处理函数：

	set_exception_handler(function($e)
	{
	    System\Error::handle($e);
	});

	set_error_handler(function($number, $error, $file, $line)
	{
	    System\Error::handle(new ErrorException($error, 0, $number, $file, $line));
	});

	register_shutdown_function(function()
	{
	    if ( ! is_null($error = error_get_last()))
	    {
	        System\Error::handle(new ErrorException($error['message'], 0, $error['type'], $error['file'], $error['line']));
	    }


在 application/config 目录下创建 error.php:

	<?php

	return array(
	    // 发生异常时显示错误细节
	    'detail' => true,

	    // 发生异常时候不记录错误日志
	    'log' => false
	);

我们通过  Error 类的 handler 方法对异常进行捕捉处理。

	<?php namespace System;

	class Error {

	    /**
	     * Error levels.
	     *
	     * @var array
	     */
	    public static $levels = array(
	        0                  => 'Error',
	        E_ERROR            => 'Error',
	        E_WARNING          => 'Warning',
	        E_PARSE            => 'Parsing Error',
	        E_NOTICE           => 'Notice',
	        E_CORE_ERROR       => 'Core Error',
	        E_CORE_WARNING     => 'Core Warning',
	        E_COMPILE_ERROR    => 'Compile Error',
	        E_COMPILE_WARNING  => 'Compile Warning',
	        E_USER_ERROR       => 'User Error',
	        E_USER_WARNING     => 'User Warning',
	        E_USER_NOTICE      => 'User Notice',
	        E_STRICT           => 'Runtime Notice'
	    );

	    /**
	     * Handle an exception.
	     *
	     * @param  Exception  $e
	     * @return void
	     */
	    public static function handle($e)
	    {
	        // 清空输出缓冲区
	        if (ob_get_level() > 0)
	        {
	            ob_clean();
	        }

	        // 获取错误级别
	        $severity = (array_key_exists($e->getCode(), static::$levels)) ? static::$levels[$e->getCode()] : $e->getCode();

	        // 获取错误文件
	        $file = $e->getFile();

	        // 格式化错误信息
	        $message = rtrim($e->getMessage(), '.');

	        // 根据配置文件决定是否记录日志
	        if (Config::get('error.log'))
	        {
	            Log::error($message.' in '.$e->getFile().' on line '.$e->getLine());
	        }


	        // 根据配置文件来决定是否显示异常信息
	        if (Config::get('error.detail'))
	        {
	            $view = View::make('error/exception')
	                ->bind('severity', $severity)
	                ->bind('message', $message)
	                ->bind('file', $file)
	                ->bind('line', $e->getLine())
	                ->bind('trace', $e->getTraceAsString())
	                ->bind('contexts', static::context($file, $e->getLine()));

	            $response = new Response($view, 500);
	            $response->send();
	        }
	        else
	        {
	            $response = new Response(View::make('error/500'), 500);
	            $response->send();
	        }

	        exit(1);
	    }

	    /**
	     * 根据错误行数获取上下文
	     *
	     * @param  string  $path
	     * @param  int     $line
	     * @param  int     $padding
	     * @return array
	     */
	    private static function context($path, $line, $padding = 5)
	    {
	        // 判断文件是否存在
	        if (file_exists($path))
	        {
	            // 读取文件， 以数组的形式存储整个文件
	            $file = file($path, FILE_IGNORE_NEW_LINES);

	            // 保证数组不为空
	            array_unshift($file, '');

	            // 计算初始偏移
	            $start = $line - $padding;

	            if ($start < 0)
	            {
	                $start = 0;
	            }

	            // 计算上下文长度
	            $length = ($line - $start) + $padding + 1;

	            if (($start + $length) > count($file) - 1)
	            {
	                $length = null;
	            }

	            return array_slice($file, $start, $length, true);
	        }

	        return array();
	    }

	}

核心是根据 php 提供的 exception 提供的信息进行封装呈现。


整个框架项目结构形如：

* application
	* |- config
		* |-applicationi.php
		* |-error.php
	* |- views
		* |-home
			* index.php
	* |- routes.php
* public
	* |- index.php
* system
	* |- config.php
	* |- error.php
	* |- loader.php
	* |- log.php
	* |- request.php
	* |- response.php
	* |- route.php
	* |- router.php
	* |- view.php