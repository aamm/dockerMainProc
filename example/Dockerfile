FROM ubuntu:14.04
ADD daemonScript /
ADD dockerMainProc /
RUN chmod u+x /daemonScript
ENTRYPOINT [ "/daemonScript" ]
