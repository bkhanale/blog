## GSoC'19: Coding Phase 2 ends

So here I am writing this blog after an exciting phase of Google Summer of Code 2019. This was the second phase of the program and it has been quite exciting, to be honest. There was a lot of research, a lot of thinking and the work to get the bears done. Let’s discuss some of the things I really enjoyed during this phase.

## RequirementsCheckBear

Probably the most exciting bear I’ve ever written. This bear does some magic. Yes, its true, magic. Let me explain:

We always have those requirements in the requirements.txt which we have to install every time we want some software to work. Well, those requirements aren’t always good. They just break, a lot of times. Yeah, I know we pin them to a specific version but for development of projects which need the most updated versions of every package, they are kept in compatibility mode. This means that they can still update, it just makes sure that they are compatible with the version we tested them on. And as the requirement progresses, they keep changing their dependencies and for as good as pip is, they create conflicts!

The conflicts are too common, and pip still doesn’t have its most required feature, the conflict resolution mechanism. It is still in development, but I think it's still gonna take some time to be good enough.

About the bear now:

The smart RequirementsCheckBear basically uses the functionality of pip-compile which comes as a part of pip-tools . That and some processing from our side makes the bear complete. The aim of the bear was to check for such conflicts, and if there are some conflicts then let the user know about it. The pip-compile is a wonderful tool, which basically mimics the installation of pip requirements. It just does it without actually doing it. So, without actually installing the requirements it generates the dependency tree. How cool is that! I tried going through the pip-tools code for the compilation part, but let's leave that for now.

If you’re curious like me, here is the code for the RequirementsCheckBear:

```py
import os.path

from coalib.bears.GlobalBear import GlobalBear
from coalib.results.Result import Result, RESULT_SEVERITY
from sarge import capture_both

from dependency_management.requirements.PipRequirement import PipRequirement


class RequirementsCheckBear(GlobalBear):
    """
    The bear to check and find any conflicting dependencies.
    """
    LANGUAGES = {
        'Python Requirements',
        'Python 2 Requirements',
        'Python 3 Requirements',
    }
    REQUIREMENTS = {PipRequirement('pip-tools', '3.8.0')}
    AUTHORS = {'The coala developers'}
    AUTHORS_EMAILS = {'coala-devel@googlegroups.com'}
    LICENSE = 'AGPL-3.0'

    def run(self, req_files: tuple):
        """
        :param req_files:
            Tuple of requirements files.
        """
        data = ''
        orig_file = ''

        for req_file in req_files:
            if not os.path.isfile(os.path.abspath(req_file)):
                raise ValueError('The file \'{}\' doesn\'t exist.'
                                 .format(req_file))

            with open(req_file) as req:
                cont = req.read()
                if not orig_file:
                    orig_file = cont
                else:
                    data += cont

        with open(req_files[0], 'a+') as temp_file:
            temp_file.write(data)

        out = capture_both('pip-compile {} -r -n --no-annotate --no-header '
                           '--no-index --allow-unsafe'.format(req_files[0]))

        if out.stderr.text and not out.stdout.text:
            lines = out.stderr.text.splitlines()
            yield Result(self,
                         message=[s for s in lines if 'Could not' in s][0],
                         severity=RESULT_SEVERITY.MAJOR,
                         )

        with open(req_files[0], 'w+') as temp_file:
            temp_file.write(orig_file)
```

Yes, the tests are also there but I don’t think you’d be interested in that. Now, this bear is just awesome because it actually helps you to control the conflicts caused due to the requirements. This bear was I think a much difficult task to do, considering the complexity of the raw solution. It’s much harder to actually complete the solution than to just think about it.

Enough, let's move to the next bear quickly:

## RegexLintBear

The regex, yeah they are quite powerful. And its equally important to keep them in a check. Sometimes, we do stupid things in the regex, we keep something we might not actually need or something which is different from what we think it is. To help with the cause, we now have a nice RegexLintBear.

This bear internally uses the regexlint tool but makes it complete and more powerful. The regexlint requires regex or strings directly as the input. But, as we know its terrible to manually pass all those strings to it. We want it to automatically get all the regex from a file and run the regexlint on each one of them and return the result. Well, the RegexLintBear exactly does this. It uses AnnotationBear which is another cool coala bear btw, to get all the strings from the files. Later we detect all the regexes out of it and run regexlint on them. Giving away the nice coala results.

Well actually, regexlint does some really nice stuff, I’m not going deep into it. But because of the bear, the tool gets much more interesting and useful

Here is some code for the nerds:

```py
import re

from queue import Queue
from sarge import run, Capture
from contextlib import suppress

from bears.general.AnnotationBear import AnnotationBear

from coalib.bears.LocalBear import LocalBear
from coalib.results.Result import Result
from coalib.settings.Section import Section
from coalib.settings.Setting import Setting
from coalib.testing.LocalBearTestHelper import execute_bear

from dependency_management.requirements.PipRequirement import PipRequirement


class RegexLintBear(LocalBear):
    LANGUAGES = {'All'}
    REQUIREMENTS = {PipRequirement('regexlint', '1.6')}
    AUTHORS = {'The coala developers'}
    AUTHORS_EMAILS = {'coala-devel@googlegroups.com'}
    LICENSE = 'AGPL-3.0'
    CAN_DETECT = {'Formatting'}

    def run(self, filename, file, language: str):
        """
        Bear for linting regex through regexlint.

        :param language:
            The programming language of the file(s).
        """
        section = Section('')
        section.append(Setting('language', language))
        bear = AnnotationBear(section, Queue())

        with execute_bear(bear, filename, file) as result:
            for src_range in result[0].contents['strings']:
                src_line = src_range.affected_source({filename: file})[0]
                regex = src_line[src_range.start.column:src_range.end.column-1]
                with suppress(re.error):
                    re.compile(regex)
                    out = run('regexlint --regex "{}"'.format(regex),
                              stdout=Capture()).stdout.text
                    if out[-3:-1] != 'OK':
                        yield Result.from_values(
                            origin=self,
                            message=out,
                            file=filename,
                            line=src_range.start.line,
                            column=src_range.start.column,
                            end_line=src_range.end.line,
                            end_column=src_range.end.column,
                        )
```

So with that, I think its time to end this blog. I’m quite happy to finish all the assigned tasks this phase. And I’m really excited about the next phase now, a lot of more interesting things to come up next month.

Also, my college is gonna start soon, so I think its gonna be quite hectic. But, nevertheless, it's going to be a lot more interesting.
