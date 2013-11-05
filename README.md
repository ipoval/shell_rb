Basic Shell Implementation in Ruby

```ruby
#!/usr/bin/env ruby

# encoding: utf-8

if __FILE__ == $0

require 'shellwords'

def main
  loop do
    $stdout.print ENV['PROMPT']

    line = exit_if_eof $stdin.gets

    commands = split_on_pipes line

    placeholder_in, placeholder_out = $stdin, $stdout
    pipe = []

    commands.each_with_index do |command, index|
      program, *arguments = Shellwords.shellsplit command

      if builtin?(program)
        call_builtin program, *arguments
      else
        if index + 1 < commands.size
          pipe = IO.pipe
          # write-only end of the pipe
          placeholder_out = pipe.last
        else
          # The last command in the pipe line so set STDOUT to $stdout of the terminal
          placeholder_out = $stdout
        end

        spawn_program program, *arguments, placeholder_out, placeholder_in

        # close references to IO objects for proper communication btw. pipes
        placeholder_out.close unless placeholder_out == $stdout
        placeholder_in.close unless placeholder_in == $stdin

        placeholder_in = pipe.first
      end
    end

    Process.waitall
  end  
end

def split_on_pipes line
  line.scan(/([^"'|]+)|["']([^"']*)["']/).flatten.compact
end

def exit_if_eof line
  line ? line.strip! : exit
end

def spawn_program program, *arguments, placeholder_out, placeholder_in
  fork {
    unless placeholder_out == $stdout
      # set STDOUT of the process to write-only end of the pipe
      $stdout.reopen(placeholder_out)
      placeholder_out.close
    end

    unless placeholder_in == $stdin
      # set STDIN of the process to read-only end of the pipe
      $stdin.reopen(placeholder_in)
      placeholder_in.close
    end

    exec program, *arguments
  }
end

def builtin?(program)
  !!SHELL_BUILT_IN_COMMANDS[program]
end

ENV['PROMPT'] = '-> '

SHELL_BUILT_IN_COMMANDS = {
  'export' => lambda { |args|
    key, value = args.split '='
    ENV[key] = value
  },
  'cd'     => lambda { |dir| Dir.chdir dir },
  'exit'   => lambda { |code = 0| exit Integer(code) },
  'exec'   => lambda { |*command| exec *command }
}

main

end
```