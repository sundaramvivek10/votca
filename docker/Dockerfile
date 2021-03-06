FROM votca/buildenv:latest

ARG TESTOPTS
ARG COVERAGE
ARG CC
ARG CXX
ARG CXXFLAGS
ARG CMAKE_BUILD_TYPE
ARG TESTING
ARG DOXYGEN
ARG DOXYGEN_COVERAGE
ARG MINIMAL
ARG MODULE_BUILD
ARG BUILD_GROMACS
ARG REGRESSION_TESTING
ARG CLANG_FORMAT
ARG CLANG_TIDY
ARG CMAKE_GENERATOR

#for coverage
ARG CI
ARG TRAVIS
ARG TRAVIS_BRANCH
ARG TRAVIS_JOB_NUMBER
ARG TRAVIS_PULL_REQUEST
ARG TRAVIS_JOB_ID
ARG TRAVIS_TAG
ARG TRAVIS_REPO_SLUG
ARG TRAVIS_COMMIT
ARG TRAVIS_OS_NAME

COPY votca/ /home/votca/votca
RUN rm -rf /home/votca/.ccache
COPY ccache/ /home/votca/.ccache
USER root
RUN chown -R votca:votca /home/votca/votca /home/votca/.ccache
USER votca

WORKDIR /home/votca/votca
RUN mkdir build
WORKDIR build
RUN ccache -z
RUN cmake -DCMAKE_VERBOSE_MAKEFILE=ON -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} \
  ${CMAKE_GENERATOR:+-G${CMAKE_GENERATOR}} \
  -DBUILD_CSGAPPS=ON -DBUILD_XTP=ON -DCMAKE_INSTALL_PREFIX=/usr \
  -DENABLE_TESTING=${TESTING} -DBUILD_CSG_MANUAL=ON -DENABLE_WERROR=ON \
  ${COVERAGE:+-DENABLE_COVERAGE_BUILD=ON} -DBUILD_OWN_GROMACS=${BUILD_GROMACS} \
  -DENABLE_REGRESSION_TESTING=${REGRESSION_TESTING} \
  ${CLANG_TIDY:+-DCMAKE_CXX_CLANG_TIDY="clang-tidy;-header-filter='.*';-checks='-*,modernize-use-nullptr,modernize-use-override,modernize-use-equals-default,modernize-use-using'"} \
  -DMODULE_BUILD=${MODULE_BUILD} ${MODULE_BUILD:+-DCMAKE_INSTALL_PREFIX=/home/votca/votca -DVOTCA_TEST_OPTS="${TESTOPTS}"} \
  ${MINIMAL:+-DCMAKE_DISABLE_FIND_PACKAGE_HDF5=ON -DCMAKE_DISABLE_FIND_PACKAGE_FFTW3=ON -DCMAKE_DISABLE_FIND_PACKAGE_MKL=ON -DBUILD_MANPAGES=OFF -DWITH_GMX=OFF -DBUILD_XTP=OFF} \
  ..
RUN cmake --build . -- -j2
RUN ccache -s
RUN if [ ${DOXYGEN} ]; then \
      cmake --build . --target doxygen; \
      if [ ${DOXYGEN_COVERAGE} ]; then \
        python -m coverxygen --xml-dir share/doc/xml/ --src-dir .. --output doxygen.coverage.info; \
        codecov -R ${PWD}/../$([ ${TRAVIS_REPO_SLUG#*/}  = votca ] || echo ${TRAVIS_REPO_SLUG#*/}) \
          -C "${TRAVIS_PULL_REQUEST_SHA:-$TRAVIS_COMMIT}" \
          -F doxygen -X gcov -f doxygen.coverage.info; \
        rm -f doxygen.coverage.info; \
      fi; \
    fi
RUN if [ ${TESTING} = ON ]; then ctest --output-on-failure ${TESTOPTS}; fi
RUN if [ ! ${MODULE_BUILD} ]; then \
      DESTDIR=${PWD}/install cmake --build . --target install && rm -rf ${PWD}/install/usr && rmdir ${PWD}/install; \
    fi
RUN if [ ${COVERAGE} ]; then \
  if [ ${CC} = clang ]; then \
    codecov -R ${PWD}/../$([ ${TRAVIS_REPO_SLUG#*/}  = votca ] || echo ${TRAVIS_REPO_SLUG#*/}) \
      -C "${TRAVIS_PULL_REQUEST_SHA:-$TRAVIS_COMMIT}" \
      -F "${CC}" -X "gcovout" -x "llvm-cov gcov"; \
  else \
    codecov -R ${PWD}/../$([ ${TRAVIS_REPO_SLUG#*/}  = votca ] || echo ${TRAVIS_REPO_SLUG#*/}) \
      -C "${TRAVIS_PULL_REQUEST_SHA:-$TRAVIS_COMMIT}" \
      -F "${CC}" -X "gcovout"; \
  fi; \
fi > codecov.log 2>&1 && tail -n 25 codecov.log || { cat codecov.log; false; }
RUN if [ -n "${CLANG_FORMAT}" ]  && [ ${TRAVIS_BRANCH} != next ]; then \
      cmake --build . --target format && \
      git -C ${PWD}/../$([ ${TRAVIS_REPO_SLUG#*/}  = votca ] || echo ${TRAVIS_REPO_SLUG#*/}) diff --submodule=diff --exit-code; \
    fi
RUN sudo cmake --build . --target install
